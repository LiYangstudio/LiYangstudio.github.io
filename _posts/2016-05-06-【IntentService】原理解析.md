
---
layout: post
title:【IntentService】原理解析
desc: 我的博客系统介绍
keywords: 'blog'
date: 2016-11-07T00:00:00.000Z
categories:
- life
tags:
- life
icon: icon-life
---

<h1>为何要用IntentService</h1>
IntentService主要是用来处理耗时的任务。 那么可能有的人会说，为何我不使用开一个线程或者其他来方式来实现呢？
<!-- more -->
因为IntentService继承自Service，属于四大组件之一。而根据Android系统的特点，系统倾向于释放没有活跃的四大组件的App。 所以当我们进行一些比较重要的耗时任务时，这个是我们一个很好的选择。 比如下载一个更新包，升级我们的Apk。  

如果当用户点击下载之后，而将App放入后台，这时候如果内存紧缺，系统就会倾向结束这个App来释放内存。 那么这个时候，由于我们有活动的IntentService，优先级较高，就不会被轻易结束。

<h1>IntentService是如何工作的？</h1>
看看它创建的代码：
```
    @Override
    public void onCreate() {
        // TODO: It would be nice to have an option to hold a partial wakelock
        // during processing, and to have a static startService(Context, Intent)
        // method that would launch the service & hand off a wakelock.

        super.onCreate();
        HandlerThread thread = new HandlerThread("IntentService[" + mName + "]");
        thread.start();

        mServiceLooper = thread.getLooper();
        mServiceHandler = new ServiceHandler(mServiceLooper);
    }
```
首先初始化了一个HandlerThread，然后执行了它的start()方法。

这个类帮我们自动创建了一个Looper，方便我们使用Handler。

看看这个类的run方法：
```
    @Override
    public void run() {
        mTid = Process.myTid();
        Looper.prepare();
        synchronized (this) {
            mLooper = Looper.myLooper();
            notifyAll();
        }
        Process.setThreadPriority(mPriority);
        onLooperPrepared();
        Looper.loop();
        mTid = -1;
    }
```
明白了之后我们继续看看，onCreate()后面的代码：
```
    mServiceLooper = thread.getLooper();
    mServiceHandler = new ServiceHandler(mServiceLooper);
```
我们来看看这个ServiceHandler的代码：
```
    private final class ServiceHandler extends Handler {
        public ServiceHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            onHandleIntent((Intent)msg.obj);
            stopSelf(msg.arg1);
        }
    }
```
至此也就是说，这个Hanlder是在一个新的线程里面（HanlderThread本质是个Thread）来处理消息的（handleMessage方法）

看看这个handleMessage方法里面有：
```
onHandleIntent((Intent)msg.obj);
```
是不是似曾相识，这个就是我们继承IntentService必须重写的方法，即在这里面进行耗时任务。

然后我们看到：
```
stopSelf(msg.arg1);
```
我们知道IntentService在执行完任务之后，就会尽快"自杀"。 从上面这句代码就可以看出来了。

OK，那我们现在来走一遍他的工作流程。
在IntentService执行完onCreate之后，就会执行：onStartCommand 

然后我们看到它的代码：
```
    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        onStart(intent, startId);
        return mRedelivery ? START_REDELIVER_INTENT : START_NOT_STICKY;
    }
    
    @Override
    public void onStart(Intent intent, int startId) {
        Message msg = mServiceHandler.obtainMessage();
        msg.arg1 = startId;
        msg.obj = intent;
        mServiceHandler.sendMessage(msg);
    }
```
发现又调用了onStart。

那这是就是我们平时使用Hanlder的场景了，通过sendMessage(msg)。然后在Handler中的handleMessage来处理消息了。

刚刚我们也看到了，在handleMessage中又调用了onHandleIntent。 

并且这个Handler是运行在一个子线程中的，所以onHandleIntent也是运行在子线程中，这就是为什么IntentService可以用来执行耗时任务的原因了！

所以它的工作流程就很清晰地呈现出来了！

-Hans 2016.2.26 23:10
