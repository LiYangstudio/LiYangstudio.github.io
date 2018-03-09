

<h1>主要组成概述：Hanlder、MessageQueue以及Looper</h1>
1.Handler：线程切换

2.MessageQueue：用来存储信息 内部实现是单链表而不是队列。

3.Looper：无限循环的方式去查找是否有消息
> 备注：Looper中一个ThreadLocal的概念
> 作用：是来存储每一个线程中数据。
> 用途：Hanlder创建的时候会采用当前线程的Looper来构造消息循环系统，Handler正式通过ThreadLocal来获取当前的线程Looper的。

<!-- more -->
<h1>源代码</h1>>

<h2>1.Handler是如何将三者联系起来的？</h2>

看看Handler创建源代码：
```
public Handler(Callback callback, boolean async) {
        ...//省略
        mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }
```
到调用了
```
	Looper.myLooper()
```
我们点进看看：
```
   /**
     * Return the Looper object associated with the current thread.  Returns
     * null if the calling thread is not associated with a Looper.
     */
    public static Looper myLooper() {
        return sThreadLocal.get();
    }
```
没错，的确是这样印证了Handler、Looper的关系。

那么MessageQueue是如何一起协同工作呢？

我们看到
```
	mQueue = mLooper.mQueue;
```
所以从不难推断出MessageQueue在Looper初始化的时候也一并初始化了。

从这里我们也可以看出：**无论我们在主线程创建多少个Handler，他们公用一个消息队列的**
<h2>2.Looper是如何工作的？</h2>
上面概述的时候我们知道，Looper是负责从消息队列中读取消息，然后回传到handler的handleMessage方法来处理消息的。

大家不知道还记不记得，我们是如何在子线程创建一个Handler的？ 我们需要调用Looper.prepare()来准备一个Looper，并且调用Looper.loop()。

很明显Looper.loop()就是无限读取消息队列的奥秘所在。

源代码(省略了部分)：
```
/**
     * Run the message queue in this thread. Be sure to call
     * {@link #quit()} to end the loop.
     */
    public static void loop() {
        final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue;

        ...

        for (;;) {
            Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }

           ...

            msg.target.dispatchMessage(msg);

            ...

           
        }
    }

    /**
     * Return the Looper object associated with the current thread.  Returns
     * null if the calling thread is not associated with a Looper.
     */
    public static Looper myLooper() {
        return sThreadLocal.get();
    }
```
第一步：先调用了Looper.myLooper()这个方法，返回当前线程所在的Looper实例。**备注：这里也刚好可以看到了ThreadLocal的用途了。**

第二步：无限死循环 如果有消息来了就：
```
	msg.target.dispatchMessage(msg);
```

这个msg.target是什么呢？ 通过Message源代码发现 就是发送这个msg对象的Handler实例： 
```
	Handler target;
```

那msg是什么时候保存的呢？通过Handler源代码发现：
```
    private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }
```
msg在入队前，保存了这个handler实例的引用。

然后调用了这个Hanlder实例的dispatchMessage的方法。

那我们来看看Handler的dispatchMessage做了什么：
```
    /**
     * Handle system messages here.
     */
    public void dispatchMessage(Message msg) {
        if (msg.callback != null) {
            handleCallback(msg);
        } else {
            if (mCallback != null) {
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            handleMessage(msg);
        }
    }
```
我们看到一句很重点的：**handleMessage(msg);**
实际上他是一个空方法：
```
    public void handleMessage(Message msg) {
    }
```
这个就是为什么我们创建Handler之后要重写这个方法来处理消息了。


