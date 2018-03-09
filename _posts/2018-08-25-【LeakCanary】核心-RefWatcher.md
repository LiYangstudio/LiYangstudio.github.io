
---
layout: post
title:【LeakCanary】核心-RefWatcher.md
desc: 我的博客系统介绍
keywords: 'blog'
date: 2016-11-07T00:00:00.000Z
categories:
- life
tags:
- life
icon: icon-life
---
## 概述

从前面的文章了解到，LeakCanary的核心就是`RefWather`的`watch`方法。所以接下来我们来分析一下`RefWatcher`类

<!-- more -->


## 关键参数与构造方法

```

public final class RefWatcher {
    public static final RefWatcher DISABLED;
    private final Executor watchExecutor;//在线程中观察对象的回收行为
    private final DebuggerControl debuggerControl;
    private final GcTrigger gcTrigger;//触发GC的包装类
    private final HeapDumper heapDumper;//打印出Heap.prof文件，用于后续分析的关键类
    private final Set<String> retainedKeys;//CopyOnWriteArraySet并发的Set集合，用于保存引用的KEY(用于快速判断对象是否已经被回收，因为Queue没有直接查询的Api)
    private final ReferenceQueue<Object> queue;//引用队列，对于判定对象是否被回收了的关键
    private final Listener heapdumpListener;//回调到DisplayLeakService的关键接口
    private final ExcludedRefs excludedRefs;

    public RefWatcher(Executor watchExecutor, DebuggerControl debuggerControl, GcTrigger gcTrigger, HeapDumper heapDumper, Listener heapdumpListener, ExcludedRefs excludedRefs) {
        this.watchExecutor = (Executor)Preconditions.checkNotNull(watchExecutor, "watchExecutor");
        this.debuggerControl = (DebuggerControl)Preconditions.checkNotNull(debuggerControl, "debuggerControl");
        this.gcTrigger = (GcTrigger)Preconditions.checkNotNull(gcTrigger, "gcTrigger");
        this.heapDumper = (HeapDumper)Preconditions.checkNotNull(heapDumper, "heapDumper");
        this.heapdumpListener = (Listener)Preconditions.checkNotNull(heapdumpListener, "heapdumpListener");
        this.excludedRefs = (ExcludedRefs)Preconditions.checkNotNull(excludedRefs, "excludedRefs");
        this.retainedKeys = new CopyOnWriteArraySet();
        this.queue = new ReferenceQueue();
    }
//....省略代码
}

```

* watchEecutor：子线程池

* gcTrigger：触发GC（一个简单的类）

* HeadpDumper：打印Headp.prof文件的类

* retainerKeys：一个CopyOnWriteArraySet支持并发的Set集合。用于保存对象的key，目的是用于快速判断对象是否已经被回收

* queue：引用队列，用于判断对象是否被回收的的关键

* headdumpListener：回调接口

 

## ReferenceQueue与引用

详情见：

[HansZone - 【Java基础】浅析Java四大引用与ReferenceQueue](http://www.hanszone.tech/2017/03/05/【Java基础】浅析Java四大引用与ReferenceQueue/)

 

实际上`RefWatcher`的核心原理就是利用了该思想

 

## RefWatcher核心方法

* watch：创建`WeakReference`对象，标记唯一的key，并且放入到队列中去

* ensureGone：用于判断对象是否被回收，并且如果一直没有被回收，则调用其他类去打印Heap内存文件，并且交由外部去分析

* gone：判断使用`Set`集合保存的key是否存在，目的是快速判断对象是否已经被回收

* removeWeaklyReachableReferences：讲已经被回收的对象的key从`Set`集合中移除

 

下面是源代码：

```

public void watch(Object watchedReference) {
    this.watch(watchedReference, "");
}

public void watch(Object watchedReference, String referenceName) {
    Preconditions.checkNotNull(watchedReference, "watchedReference");
    Preconditions.checkNotNull(referenceName, "referenceName");
    if(!this.debuggerControl.isDebuggerAttached()) {
        final long watchStartNanoTime = System.nanoTime();
        String key = UUID.randomUUID().toString();//生成观察对象的key
        this.retainedKeys.add(key);//保存key（用于后续快速判断对象是否被从队列移除了，因为queue没有快速查询的api）
        final KeyedWeakReference reference = new KeyedWeakReference(watchedReference, key, referenceName, this.queue);//创建一个特殊的弱应用对象（里面保存了key，和指定了自己的queue）
        this.watchExecutor.execute(new Runnable() {//放入线程池去观察
            public void run() {
                RefWatcher.this.ensureGone(reference, watchStartNanoTime);//在线程中观察回收的对象
            }
        });
    }
}

void ensureGone(KeyedWeakReference reference, long watchStartNanoTime) {
    long gcStartNanoTime = System.nanoTime();
    long watchDurationMs = TimeUnit.NANOSECONDS.toMillis(gcStartNanoTime - watchStartNanoTime);
    this.removeWeaklyReachableReferences();//将已在queue中的对象，从我们的set（保存key）中移除。表明该对象已经被回收了
    if(!this.gone(reference) && !this.debuggerControl.isDebuggerAttached()) {//判断该对象是否已经被回收
        this.gcTrigger.runGc();//执行一次gc
        this.removeWeaklyReachableReferences();//再次执行，将已在queue中的对象，从我们的set（保存key）中移除。表明该对象已经被回收了
        if(!this.gone(reference)) {//如果还没有被回收，说明有可能已经发生了内存泄漏（因为这个在Android中，观察的是Activity，并且是在onDestory之后才进行观察的，所以如果没有被回收。极有可能是发生了内存泄漏）
            long startDumpHeap = System.nanoTime();
            long gcDurationMs = TimeUnit.NANOSECONDS.toMillis(startDumpHeap - gcStartNanoTime);
            File heapDumpFile = this.heapDumper.dumpHeap();//保存现在当时的heap文件
            if(heapDumpFile == HeapDumper.NO_DUMP) {
                return;
            }

            long heapDumpDurationMs = TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - startDumpHeap);
            this.heapdumpListener.analyze(new HeapDump(heapDumpFile, reference.key, reference.name, this.excludedRefs, watchDurationMs, gcDurationMs, heapDumpDurationMs));//调用外部去分析这个heap文件
        }

    }
}

private boolean gone(KeyedWeakReference reference) {
    return !this.retainedKeys.contains(reference.key);//set集合的用处，快速判断这个对象是否被回收了
}

private void removeWeaklyReachableReferences() {
    KeyedWeakReference ref;
    while((ref = (KeyedWeakReference)this.queue.poll()) != null) {//将已经被回收的对象从我们的set集合中移除
        this.retainedKeys.remove(ref.key);
    }

}

```

小结：
* 要始终记住，`watch`的对象是Android中具有生命周期的`Activity`；并且是`在Application`注册观察在`Activity`执行了`onDestory`方法之后，才进行观察的

* 基于第一点，所以在方法内，两次进行`removeWeaklyReachableReferences`方法和GC。目的是为了怀疑该`Activity`是否被及时回收

* 如果没有被回收，则需要分析Heap文件，判定其是否真的出现内存泄漏

