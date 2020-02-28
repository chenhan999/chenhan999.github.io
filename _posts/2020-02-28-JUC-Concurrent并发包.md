---
layout:     post
title:      JUC并发包学习
subtitle:   JUC并发包
date:       2020-02-28
author:     Han
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - JUC-并发包
---
学习背景：

在之前report导出需求中，因为数据量较大，导出时间太久，故使用到了多线程查询，期间接触到JUC，在此整理一下学习内容

java.util.concurrent并发编程包是专门为Java并发编程设计的，其中设计的类主要分为以下几部分：

1. 显式锁

2. 原子变量

3. 线程池

4. 并发容器

5. 同步工具类

***
### 并发容器-> CopyOnWriteArrayList

问题现象：

使用多线程处理数据时，最终导出数据会有数据缺失的现象，通过排查代码发现是因为ArrayList是线程不安全的容器，
在多线程情况下会出现多个线程操作同一个元素的现象，导致数据被覆盖。

解决方法：

1.通过 Collections 的 synchronizedList 方法将 ArrayList 转换成线程安全的容器后再使用。

`List<Object> list =Collections.synchronizedList(new ArrayList<Object>);`

2.list.add()方法加锁

```
synchronized(list.get()) {
 list.get().add(model);
 }
 ```
3.使用线程安全的 CopyOnWriteArrayList 代替线程不安全的 ArrayList。
 
 ```
 List<Object> list1 = new CopyOnWriteArrayList<Object>();
 ```
关于CopyOnWriteArrayList：
 
* CopyOnWriteArrayList是线程安全容器(相对于ArrayList)，底层由ReentrantLock锁和复制数组的方式来实现线程安全。

```
   public boolean add(E e) {
        // 加锁
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            // 得到原数组的长度和元素
            Object[] elements = getArray();
            int len = elements.length;
            
            // 复制出一个新数组
            Object[] newElements = Arrays.copyOf(elements, len + 1);
            
            // 添加时，将新元素添加到新数组中
            newElements[len] = e;
            
            // 将volatile Object[] array 的指向替换成新数组
            setArray(newElements);
            return true;
        } finally {
            lock.unlock();
        }
    }
```
* CopyOnWriteArrayList在遍历的使用不会抛出ConcurrentModificationException异常，并且遍历的时候就不用额外加锁
* 元素可以为null
* 缺点是比较耗费内存
* hashMap -> ConcurrentHashMap， ArraySet -> CopyOnWriteArraySet , ArrayList -> CopyOnWriteArrayList ,后者都是在JUC中针对并发

### 同步工具类-> CountDownLatch

在report导出中，在多个线程查询的数据统一写入List，最终在主线程中通过Excel导出，调试中发现子线程还未执行完就开始执行主线程，导致数据缺失
在JUC中找到解决方法CountDownLatch类，通过  
`new CountDownLatch()`  
`countDownLatch.countDown();`  
`countDownLatch.await();`
配合使用，使主线程等待子线程全部执行完成之后再执行。  
类似与Thread的join方法，其区别在于：  
在使用join()中，多个线程只有在执行完毕之后欧才能被解除阻塞，而在CountDownLatch中，线程可以在任何时候任何位置调用countdown方法减少计数，通过这种方式，我们可以更好地控制线程的解除阻塞，而不是仅仅依赖于连接线程的完成。
代码中的使用
![avatar](/img/report.bmp)

