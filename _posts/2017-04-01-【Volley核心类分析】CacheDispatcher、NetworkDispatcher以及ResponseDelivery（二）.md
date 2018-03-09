

---
layout: 【Volley核心类分析】RequestQueue（一）
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

<h1>1.前言</h1>
上一篇文章按照自己的思路来写，对于他人来说可以能不是那么容易读懂，想要改一下却不知道如何下手。

<!-- more -->
我们先回顾一下上一篇我们的分析，我们分析到了RequestQueue的start方法，这里启动了缓存的调度线程以及网络的调度线程。 我们没有进行具体的分析，那么这节课我们来分析一下。

<h1>2.CacheDispatcher</h1>
首先看看构造方法：

```
public CacheDispatcher(
            BlockingQueue<Request<?>> cacheQueue, BlockingQueue<Request<?>> networkQueue,
            Cache cache, ResponseDelivery delivery) {
        mCacheQueue = cacheQueue;
        mNetworkQueue = networkQueue;
        mCache = cache;
        mDelivery = delivery;
    }
```
传入了阻塞的缓存任务、阻塞的网络队列、缓存类、以及结果分派的类

我们来主要看看它的run方法
```
@Override
    public void run() {
        if (DEBUG) VolleyLog.v("start new dispatcher");
		//设置优先级：
				Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);

        // Make a blocking call to initialize the cache.
        mCache.initialize();//打开缓存
		
		// 死循环
        while (true) {
            try {
                // Get a request from the cache triage queue, blocking until
                // at least one is available.
                // 阻塞直到有任务到达
                final Request<?> request = mCacheQueue.take();
                request.addMarker("cache-queue-take");

                // If the request has been canceled, don't bother dispatching it.
                //如果该任务已经取消，则放弃
                if (request.isCanceled()) {
                    request.finish("cache-discard-canceled");
                    continue;
                }

                // Attempt to retrieve this item from cache. 
                //根据key拿到缓存
                Cache.Entry entry = mCache.get(request.getCacheKey());
                //如果没有这个缓存 则放入 网络请求队列
                if (entry == null) {
                    request.addMarker("cache-miss");
                    // Cache miss; send off to the network dispatcher.
                    mNetworkQueue.put(request);
                    continue;
                }

                // If it is completely expired, just send it to the network.
                //如果已经过期，则还是放入网络请求队列。
                if (entry.isExpired()) {
                    request.addMarker("cache-hit-expired");
                    request.setCacheEntry(entry);
                    mNetworkQueue.put(request);
                    continue;
                }

                // We have a cache hit; parse its data for delivery back to the request.
                request.addMarker("cache-hit");
                //如果有缓存，则解析成我们的response
                Response<?> response = request.parseNetworkResponse(
                        new NetworkResponse(entry.data, entry.responseHeaders));
                request.addMarker("cache-hit-parsed");
				//如果不需要刷新，让mDelivery分发我们的结果回去。
                if (!entry.refreshNeeded()) {
                    // Completely unexpired cache hit. Just deliver the response.
                    mDelivery.postResponse(request, response);
                } else {
                    // Soft-expired cache hit. We can deliver the cached response,
                    // but we need to also send the request to the network for
                    // refreshing.
                    request.addMarker("cache-hit-refresh-needed");
                    request.setCacheEntry(entry);

                    // Mark the response as intermediate.
                    response.intermediate = true;

                    // Post the intermediate response back to the user and have
                    // the delivery then forward the request along to the network.
                    //如果需要刷新，则放到网络任务队列，并开始执行
                    mDelivery.postResponse(request, response, new Runnable() {
                        @Override
                        public void run() {
                            try {
                                mNetworkQueue.put(request);
                            } catch (InterruptedException e) {
                                // Not much we can do about this.
                            }
                        }
                    });
                }

            } catch (InterruptedException e) {
                // We may have been interrupted because it was time to quit.
                if (mQuit) {
                    return;
                }
                continue;
            }
        }
    }
```
大家可以稍微看看英文和我的中文解释，相信大家都很容易看懂。这里面做了什么。 

不过让我更加兴奋的时候，如何中断一个线程。 自从Java弃用了thread.stop()方法，我在一段时间内不知道如何立刻安全地结束这个线程。

现在我们可以参照这里的。如果将来我们要中断一个线程任务，我们就可以这样写了。

```
MyThread t = new MyThread();
//当我们要中断的时候，可以写成
t.quit();
t.interrupt();

public class MyThread extends Thread{
	private boolean mQuit;

	public void quit(){
		mQuit = true;
	}
	public void run {
		try {
		//耗时任务
		}catch (InterruptedException e) {
           if (mQuit) {
	            return;
           }
                
	    }
	}
}
```
CacheDispatcher代码量不是很多，也不是和很难看懂，但一开始让我有点费解的是

```
if (!entry.refreshNeeded()) 
```
这里这个判断。 我不明白前面明明判断了，这个数据是否已经过期，为何这里又判断是否需要刷新呢？

然去看了一下Trinea前辈的分析。 意思是：
进行结果新鲜度检验，如果不新鲜，则将Request加入到NetworkRequest做新鲜度检验。

看完我还不是太理解，我们继续往下分析，看看能不能找到答案！

引用一下Trinea前辈的CacheDispatcher流程图：
![这里写图片描述](http://img.blog.csdn.net/20160201171815257)

<h1>3.NetworkDispatcher</h1>
看看构造方法：

```
    public NetworkDispatcher(BlockingQueue<Request<?>> queue,
            Network network, Cache cache,
            ResponseDelivery delivery) {
        mQueue = queue;
        mNetwork = network;
        mCache = cache;
        mDelivery = delivery;
    }
```
参数还是我们熟悉的参数，那我就不多做解释了，我们来看看run方法：

```
 @Override
    public void run() {
        //优先级设置
        Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
        Request<?> request;
        while (true) {
            long startTimeMs = SystemClock.elapsedRealtime();
            // release previous request object to avoid leaking request object when mQueue is drained.
            //释放之前的request对象，当我们队列退出时避免内存泄漏
            request = null;
            try {
                // Take a request from the queue.
                request = mQueue.take();
            } catch (InterruptedException e) {
                // We may have been interrupted because it was time to quit.
                if (mQuit) {
                    return;
                }
                continue;
            }

            try {
                request.addMarker("network-queue-take");

                // If the request was cancelled already, do not perform the
                // network request.
                if (request.isCanceled()) {
                    request.finish("network-discard-cancelled");
                    continue;
                }
				
                addTrafficStatsTag(request);

                // Perform the network request.
                
                //执行网络请求任务
                NetworkResponse networkResponse = mNetwork.performRequest(request);
                request.addMarker("network-http-complete");

                // If the server returned 304 AND we delivered a response already,
				//如果重定向或者已经分派，则返回
                if (networkResponse.notModified && request.hasHadResponseDelivered()) {
                    request.finish("not-modified");
                    continue;
                }

                // Parse the response here on the worker thread.
                //解析
                Response<?> response = request.parseNetworkResponse(networkResponse);
                request.addMarker("network-parse-complete");

                // Write to cache if applicable.
                // TODO: Only update cache metadata instead of entire record for 304s.
                //是否是需要缓存
                if (request.shouldCache() && response.cacheEntry != null) {
                    mCache.put(request.getCacheKey(), response.cacheEntry);
                    request.addMarker("network-cache-written");
                }

                // Post the response back.
                request.markDelivered();
                //分发成功结果
                mDelivery.postResponse(request, response);
	           
	           // 以下都是用来返回失败结果
			 } catch (VolleyError volleyError) {
                volleyError.setNetworkTimeMs(SystemClock.elapsedRealtime() - startTimeMs);
                parseAndDeliverNetworkError(request, volleyError);
            } catch (Exception e) {
                VolleyLog.e(e, "Unhandled exception %s", e.toString());
                VolleyError volleyError = new VolleyError(e);
                volleyError.setNetworkTimeMs(SystemClock.elapsedRealtime() - startTimeMs);
                mDelivery.postError(request, volleyError);
            }
        }
    }
```
这里这段代码我也还是不太理解：

```
 // Write to cache if applicable.
 // TODO: Only update cache metadata instead of entire record for 304s.
  //是否是需要缓存
  if (request.shouldCache() && response.cacheEntry != null) {
       mCache.put(request.getCacheKey(),response.cacheEntry);
	   request.addMarker("network-cache-written");
 }
```
回去看看Trinea前辈的分析：

> 304响应，并且请求已经有响应传输（用来验证新鲜度的请求）

还记得CacheDispatcher也有出现这个新鲜度这个词语吧？ 我们暂时再放一下，看看后面能不能找到答案。

引用一下Trinea前辈的NetworkDispatcher流程图：
![这里写图片描述](http://img.blog.csdn.net/20160201171841729)

<h1>4.ResponseDelivery</h1>
实际上他是一个接口，具体实现类是ExecutorDelivery。

我们先看看ResponseDelivery代码：

```
public interface ResponseDelivery {
    /**
     * Parses a response from the network or cache and delivers it.
     */
    public void postResponse(Request<?> request, Response<?> response);

    /**
     * Parses a response from the network or cache and delivers it. The provided
     * Runnable will be executed after delivery.
     */
    public void postResponse(Request<?> request, Response<?> response, Runnable runnable);

    /**
     * Posts an error for the given request.
     */
    public void postError(Request<?> request, VolleyError error);
}
```
我们需要注意一下这个：

```
    /**
     * Parses a response from the network or cache and delivers it. The provided
     * Runnable will be executed after delivery.
     */
    public void postResponse(Request<?> request, Response<?> response, Runnable runnable);
```
Runnable will be executed after delivery. 再分派玩结果之后，这个Runnable要被执行时什么意思？ 还记得我们的CacheDispatcher在新鲜度的那个if判断：

```
//如果不需要刷新，让mDelivery分发我们的结果回去。
                if (!entry.refreshNeeded()) {
                    // Completely unexpired cache hit. Just deliver the response.
                    mDelivery.postResponse(request, response);
                } else {
                    // Soft-expired cache hit. We can deliver the cached response,
                    // but we need to also send the request to the network for
                    // refreshing.
                    request.addMarker("cache-hit-refresh-needed");
                    request.setCacheEntry(entry);

                    // Mark the response as intermediate.
                    response.intermediate = true;

                    // Post the intermediate response back to the user and have
                    // the delivery then forward the request along to the network.
                    //如果需要刷新，则放到网络任务队列，并开始执行
                    mDelivery.postResponse(request, response, new Runnable() {
                        @Override
                        public void run() {
                            try {
                                mNetworkQueue.put(request);
                            } catch (InterruptedException e) {
                                // Not much we can do about this.
                            }
                        }
                    });
```
else的里面调用这个3个参数的postResponse。难不成我们的新鲜度能够在ResponseDelivery这里揭开谜底？

我们现在来看看他的具体实现类ExecutorDelivery的源代码。

看看构造方法：

```
/**
 * Delivers responses and errors.
 */
public class ExecutorDelivery implements ResponseDelivery {
    /** Used for posting responses, typically to the main thread. */
    private final Executor mResponsePoster;

    /**
     * Creates a new response delivery interface.
     * @param handler {@link Handler} to post responses on
     */
    public ExecutorDelivery(final Handler handler) {
        // Make an Executor that just wraps the handler.
        mResponsePoster = new Executor() {
            @Override
            public void execute(Runnable command) {
                handler.post(command);
            }
        };
    }

    /**
     * Creates a new response delivery interface, mockable version
     * for testing.
     * @param executor For running delivery tasks
     */
    public ExecutorDelivery(Executor executor) {
        mResponsePoster = executor;
    }
    ...
}

```
还记得我们这个Handler创建的时候是对应主线程的吗？我们看看之前RequestQueue创建的时候的代码：

```
  public RequestQueue(Cache cache, Network network, int threadPoolSize) {
        this(cache, network, threadPoolSize,
                new ExecutorDelivery(new Handler(Looper.getMainLooper())));
    }
```
这里的Looper.getMainLooper()，我们看看getMainLooper是什么：

```
    /** Returns the application's main looper, which lives in the main thread of the application.
     */
    public static Looper getMainLooper() {
        synchronized (Looper.class) {
            return sMainLooper;
        }
    }
```
从注释可以看到，这个是主线程的Looper! 所以，回传的结果都是回传到主线程。

```
   @Override
    public void postResponse(Request<?> request, Response<?> response) {
        postResponse(request, response, null);
    }

    @Override
    public void postResponse(Request<?> request, Response<?> response, Runnable runnable) {
        request.markDelivered();
        request.addMarker("post-response");
        mResponsePoster.execute(new ResponseDeliveryRunnable(request, response, runnable));
    }

    @Override
    public void postError(Request<?> request, VolleyError error) {
        request.addMarker("post-error");
        Response<?> response = Response.error(error);
        mResponsePoster.execute(new ResponseDeliveryRunnable(request, response, null));
    }
```

我们来看看那三个方法的具体实现。
发现他又new ResponseDeliveryRunnable这个。我们再来看看这个内部类的代码：

```
  /**
     * A Runnable used for delivering network responses to a listener on the
     * main thread.
     */
    @SuppressWarnings("rawtypes")
    private class ResponseDeliveryRunnable implements Runnable {
        private final Request mRequest;
        private final Response mResponse;
        private final Runnable mRunnable;

        public ResponseDeliveryRunnable(Request request, Response response, Runnable runnable) {
            mRequest = request;
            mResponse = response;
            mRunnable = runnable;
        }

        @SuppressWarnings("unchecked")
        @Override
        public void run() {
            // If this request has canceled, finish it and don't deliver.
            if (mRequest.isCanceled()) {
                mRequest.finish("canceled-at-delivery");
                return;
            }

            // Deliver a normal response or error, depending.
            if (mResponse.isSuccess()) {
                mRequest.deliverResponse(mResponse.result);
            } else {
                mRequest.deliverError(mResponse.error);
            }

            // If this is an intermediate response, add a marker, otherwise we're done
            // and the request can be finished.
            if (mResponse.intermediate) {
                mRequest.addMarker("intermediate-response");
            } else {
                mRequest.finish("done");
            }

            // If we have been provided a post-delivery runnable, run it.
            if (mRunnable != null) {
                mRunnable.run();
            }
       }
    }
```
这个是配合Handler回传的主线程的Runnable类。 相信里面的代码都很好理解。但注意最后一个：

```
// If we have been provided a post-delivery runnable, run it.
 if (mRunnable != null) {
      mRunnable.run();
 }
```
意思是，如果我们已经提供了一个过去分派的Runnable对象，则执行它。

那这个到底和Trinea前辈提及的那几个新鲜度有什么关系呢？？

至此我还是没有找到这个新鲜度是什么意思。。

<h1>5.猜测</h1>
我猜测这里的应该是，先将原本的数据（旧数据）通过ResponseDelivery传到主线程进行更新。然后，再将Request加入到NetworkQueue中等待执行，更新数据。  那么为何要封装在一个Runnable对象中去处理呢？明明只是一个简单的一句话：

```
mNetworkQueue.put(request);
```
我想提出的合理解释是：有可能当前网络队列里面是空队列，如果一边加入到队列，一边将数据更新到我的View中。在网速够快的情况下，可能会重复刷新我的UI，或者造成其他不良影响？

所以这句代码，要等待原本数据回调之后，再进行请求。

当然仅仅是我的一个猜测，如果有错误的地方望多多指出！


