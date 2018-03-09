
---
layout: post
title: 【Java基础】浅析Java四大引用与ReferenceQueue.md
desc: 我的博客系统介绍
keywords: 'blog'
date: 2017-1-28T00:00:00.000Z
categories:
- life
tags:
- life
icon: fa-life
---
## Java四大引用

* 强引用：对象绝对不会被回收，直到程序退出为止（包含OOM）

* 软引用：当即将OOM时候，JVM会回收软引用对象。若空间依旧不够，抛出OOM

* 弱引用：触发GC的时候，弱引用极有可能被回收。（由于GC线程优先级低，所以不一定能够回收所有的弱引用对象）

* 虚引用：又称为幽灵引用。目前暂时不知道有何用处

<!-- more -->


## ReferenceQueue

* 其作用是：

> Reference queues, to which registered reference objects are appended by the garbage collector after the appropriate reachability changes are detected.
> 引用队列，用于检测在发生垃圾回收之后对象的可达性变更之后，讲对象放入该队列。（就是说明该对象被回收之后，该队列就会有该对象了）

* 通过查看`ReferenceQueue`的源代码发现实现很简单，又以下特点：

    * 数据结构为单项链表，节点为`Reference`（所有引用的基类）

    * 内部有一个`Lock`类（就是一个简单的`Object`类，没有什么特别），实现并发同步

    * 当队列为空的时候，`remove`方法返回的是`null`



## 例子描述

```

public static void main(String args[]) throws InterruptedException {
    Object object = new Object();
    ReferenceQueue<Object> queue = new ReferenceQueue<>();
    WeakReference<Object> weakReference = new WeakReference<Object>(object, queue);//创建弱引用，并且指定队列queuee
    //object = null;
    Thread.sleep(500);//暂停500ms
    //触发GC
    System.gc();
    System.runFinalization();

    Reference rObj = null;
    while ((rObj = queue.poll()) != null) {
        System.out.println(rObj.get());
    }
}
```

过程：

* 创建一个`ReferenceQueue`

* 创建一个对象

* 创建弱引用

* 触发GC

* 遍历`ReferenceQueue`查看是否有对象



结果：

* 发现没有任何输出



结论：

* 符合我们的预期，因为在结束之前，`Object`存在引用，所以并不会被GC回收（`ReferenceQueue`为空）

* 那么要如何有输出呢？只需要将我们注释的`object = null`添加到代码，就会发现有输出了（`ReferenceQueue`不为空）

* 为什么这样就会有输出？上面讲解的时候说了，当一个对象被GC回收之后，会被自动添加到这个队列中去

