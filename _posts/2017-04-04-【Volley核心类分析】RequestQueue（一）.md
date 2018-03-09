

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
Volley ， 当我从第一次接触安卓开始就已经听过这个库的存在。 

那时候我还不懂得什么的封装，每次都憨憨地从创建HttpURLConnection开始，然后open，然后getInputstream，然后再BufferedReader一步一步的将网页返回的数据变成一个String，再Gson解析成我要的数据来展示。

<!-- more -->
一直怀揣着对Volley的神秘感。对他的认识也就是道听途说来的，所以感觉还是需要深入一下。

所以写这篇文章也是为了加深一下自己的记忆。

因为自己水平非常有限，也不是什么大神，所以只能粗浅地分析一下，要是有错误，望各位批评指正。


这里附上一片专业的源代码分析，[Volley 源码解析 - Trinea](http://a.codekk.com/detail/Android/Trinea/Volley%20%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90)

<h1>2.RequestQueue </h1>
使用Volley的第一步：是创建一个RequestQueue队列。

```
RequestQueue mQueue = Volley.newRequestQueue(context);
```
OK，那我们来看看Volley.newRequestQueue()这个方法都做了什么。

附上Volley源代码：
```
public class Volley {

	public static RequestQueue newRequestQueue(Context context) {
        return newRequestQueue(context, null);
    }
    
     public static RequestQueue newRequestQueue(Context context, HttpStack stack)
    {
    	return newRequestQueue(context, stack, -1);
    }
    
    public static RequestQueue newRequestQueue(Context context, int maxDiskCacheBytes) {
        return newRequestQueue(context, null, maxDiskCacheBytes);
    }

	public static RequestQueue newRequestQueue(Context context, HttpStack stack, int maxDiskCacheBytes) {
        File cacheDir = new File(context.getCacheDir(), DEFAULT_CACHE_DIR);

        String userAgent = "volley/0";
        try {
            String packageName = context.getPackageName();
            PackageInfo info = context.getPackageManager().getPackageInfo(packageName, 0);
            userAgent = packageName + "/" + info.versionCode;
        } catch (NameNotFoundException e) {
        }

        if (stack == null) {
            if (Build.VERSION.SDK_INT >= 9) {
                stack = new HurlStack();
            } else {
                // Prior to Gingerbread, HttpUrlConnection was unreliable.
                // See: http://android-developers.blogspot.com/2011/09/androids-http-clients.html
                stack = new HttpClientStack(AndroidHttpClient.newInstance(userAgent));
            }
        }

        Network network = new BasicNetwork(stack);
        
        RequestQueue queue;
        if (maxDiskCacheBytes <= -1)
        {
        	// No maximum size specified
        	queue = new RequestQueue(new DiskBasedCache(cacheDir), network);
        }
        else
        {
        	// Disk cache size specified
        	queue = new RequestQueue(new DiskBasedCache(cacheDir, maxDiskCacheBytes), network);
        }

        queue.start();

        return queue;
    }
}
```
我们一步一步看，发现我们的一个参数的带参构造调用了两个参数的构造：

```
    public static RequestQueue newRequestQueue(Context context, HttpStack stack)
    {
    	return newRequestQueue(context, stack, -1);
    }
```
两个参数的构造，调用了三个参数的构造：

```
    public static RequestQueue newRequestQueue(Context context, int maxDiskCacheBytes) {
        return newRequestQueue(context, null, maxDiskCacheBytes);
    }
```
最终调用了，这个三个参数的构造方法：

```
public static RequestQueue newRequestQueue(Context context, HttpStack stack, int maxDiskCacheBytes) {
        File cacheDir = new File(context.getCacheDir(), DEFAULT_CACHE_DIR);

        String userAgent = "volley/0";
        try {
            String packageName = context.getPackageName();
            PackageInfo info = context.getPackageManager().getPackageInfo(packageName, 0);
            userAgent = packageName + "/" + info.versionCode;
        } catch (NameNotFoundException e) {
        }

        if (stack == null) {
            if (Build.VERSION.SDK_INT >= 9) {
                stack = new HurlStack();
            } else {
                // Prior to Gingerbread, HttpUrlConnection was unreliable.
                // See: http://android-developers.blogspot.com/2011/09/androids-http-clients.html
                stack = new HttpClientStack(AndroidHttpClient.newInstance(userAgent));
            }
        }

        Network network = new BasicNetwork(stack);
        
        RequestQueue queue;
        if (maxDiskCacheBytes <= -1)
        {
        	// No maximum size specified
        	queue = new RequestQueue(new DiskBasedCache(cacheDir), network);
        }
        else
        {
        	// Disk cache size specified
        	queue = new RequestQueue(new DiskBasedCache(cacheDir, maxDiskCacheBytes), network);
        }

        queue.start();

        return queue;
    }
```
我们来分析一下这个

```
File cacheDir = new File(context.getCacheDir(), 	
				DEFAULT_CACHE_DIR);
```
这里创建了Volley的缓存路径

```
String userAgent = "volley/0";
	try {
	     String packageName = context.getPackageName();
	     PackageInfo info = context.getPackageManager().getPackageInfo(packageName, 0);
	     userAgent = packageName + "/" + info.versionCode;
	} catch (NameNotFoundException e) {

}
```
这里进行获取包名，然后赋值给userAgent。 

因为我还没有学计算机网络，根据自己的理解，放在请求头里面，供后台用来判断当前客户端种类的。我们可以用电脑的浏览器:ctrl+F12，随便输入一个网站，点开控制台Network，就可以看到请求头。 这也就是为什么平时我们用手机输入相同的网页，而手机却会跳转到适应手机屏幕的网页的原理。

这里附上：Trinea前辈在源代码中分析UserAgent的一段：

> (3). 关于 User Agent
通过代码我们发现如果是使用 AndroidHttpClient，Volley 还会将请求头中的 User-Agent 字段设置为 App 的 ${packageName}/${versionCode}，如果异常则使用 "volley/0"，不过这个获取 User-Agent 的操作应该放到 if else 内部更合适。而对于 HttpURLConnection 却没有任何操作，为什么呢？
如果用 Fiddler 或 Charles 对数据抓包我们会发现，我们会发现 HttpURLConnection 默认是有 User-Agent 的，类似：
Dalvik/1.6.0 (Linux; U; Android 4.1.1; Google Nexus 4 - 4.1.1 - API 16 - 768x1280_1 Build/JRO03S)
经常用 WebView 的同学会也许会发现似曾相识，是的，WebView 默认的 User-Agent 也是这个。实际在请求发出之前，会检测 User-Agent 是否为空，如果不为空，则加上系统默认 User-Agent。在 Android 2.1 之后，我们可以通过
String userAgent = System.getProperty("http.agent");
得到系统默认的 User-Agent，Volley 如果希望自定义 User-Agent，可在自定义 Request 中重写 getHeaders() 函数

```
@Override
public Map<String, String> getHeaders() throws AuthFailureError {
    // self-defined user agent
    Map<String, String> headerMap = new HashMap<String, String>();
    headerMap.put("User-Agent", "android-open-project-analysis/1.0");
    return headerMap;
}
```
OK！ 要是看不懂其实也没啥关系，我们继续往下分析。

```
        if (stack == null) {
            if (Build.VERSION.SDK_INT >= 9) {
                stack = new HurlStack();
            } else {
                // Prior to Gingerbread, HttpUrlConnection was unreliable.
                // See: http://android-developers.blogspot.com/2011/09/androids-http-clients.html
                stack = new HttpClientStack(AndroidHttpClient.newInstance(userAgent));
            }
        }

```
这里根据当前的SDK版本。若在2.3以下则是使用HttpClient进行网络请求，以上则使用HttpURLConnection。

我们可以看到

```
HttpClientStack(AndroidHttpClient.newInstance(userAgent));
```
使用了userAgent而

```
stack = new HurlStack();
```
没有使用，这就是Trinea前辈说的：

> HttpURLConnection 默认是有 User-Agent 的，类似：
Dalvik/1.6.0 (Linux; U; Android 4.1.1; Google Nexus 4 - 4.1.1 - API 16 - 768x1280_1 Build/JRO03S)

关于如何选择HttpClient与HttpURLConnection的具体原因，Trinea前辈也在文章中给出：

> (2). HttpURLConnection 和 AndroidHttpClient(HttpClient 的封装)如何选择及原因：
在 Froyo(2.2) 之前，HttpURLConnection 有个重大 Bug，调用 close() 函数会影响连接池，导致连接复用失效，所以在 Froyo 之前使用 HttpURLConnection 需要关闭 keepAlive。
另外在 Gingerbread(2.3) HttpURLConnection 默认开启了 gzip 压缩，提高了 HTTPS 的性能，Ice Cream Sandwich(4.0) HttpURLConnection 支持了请求结果缓存。
再加上 HttpURLConnection 本身 API 相对简单，所以对 Android 来说，在 2.3 之后建议使用 HttpURLConnection，之前建议使用 AndroidHttpClient。

接着我们看到这一段代码：

```
Network network = new BasicNetwork(stack);
```
这里将我们的发起网络请求的stack传入，所以我们可以猜测到，具体执行我们的request就是这个类了，那么是不是呢？


实际上这个Network是一个接口，我们来看看它的代码：
```
/**
 * An interface for performing requests.
 */
public interface Network {
    /**
     * Performs the specified request.
     * @param request Request to process
     * @return A {@link NetworkResponse} with data and caching metadata; will never be null
     * @throws VolleyError on errors
     */
    public NetworkResponse performRequest(Request<?> request) throws VolleyError;
}
```
从类的注释（An interface for performing requests.）可以看出，这就是执行requests的接口

我们看看它的performRequest方法，这个就是具体执行我们request的方法了。 它返回了一个带有数据以及缓存数据(metadata)的NetworkResponse对象。

我们来看看它的源代码：

```
/**
 * Data and headers returned from {@link Network#performRequest(Request)}.
 */
public class NetworkResponse {
	    /** The HTTP status code. */
    public final int statusCode;

    /** Raw data from this response. */
    public final byte[] data;

    /** Response headers. */
    public final Map<String, String> headers;

    /** True if the server returned a 304 (Not Modified). */
    public final boolean notModified;

    /** Network roundtrip time in milliseconds. */
    public final long networkTimeMs;
    
    /**
     * Creates a new network response.
     * @param statusCode the HTTP status code
     * @param data Response body
     * @param headers Headers returned with this response, or null for none
     * @param notModified True if the server returned a 304 and the data was already in cache
     * @param networkTimeMs Round-trip network time to receive network response
     */
    public NetworkResponse(int statusCode, byte[] data, Map<String, String> headers,
            boolean notModified, long networkTimeMs) {
        this.statusCode = statusCode;
        this.data = data;
        this.headers = headers;
        this.notModified = notModified;
        this.networkTimeMs = networkTimeMs;
    }
	....
	//还有一些其他构造方法，我们省略不看。
}
```
从这个类可以看出，这个类保存了返回的信息，如：
statusCode（状态码）、data（byte[]数据）、headers（返回的头信息）、notModified（是否重定向）、networkTimeMs（请求时间）

现在我们来看看BasicNetwork，这个是Network的具体实现类。

```
/**
 * A network performing Volley requests over an {@link HttpStack}.
 */
public class BasicNetwork implements Network {
    protected static final boolean DEBUG = VolleyLog.DEBUG;

    private static int SLOW_REQUEST_THRESHOLD_MS = 3000;

    private static int DEFAULT_POOL_SIZE = 4096;

    protected final HttpStack mHttpStack;

    protected final ByteArrayPool mPool;

    /**
     * @param httpStack HTTP stack to be used
     */
    public BasicNetwork(HttpStack httpStack) {
        // If a pool isn't passed in, then build a small default pool that will give us a lot of
        // benefit and not use too much memory.
        this(httpStack, new ByteArrayPool(DEFAULT_POOL_SIZE));
    }

    /**
     * @param httpStack HTTP stack to be used
     * @param pool a buffer pool that improves GC performance in copy operations
     */
    public BasicNetwork(HttpStack httpStack, ByteArrayPool pool) {
        mHttpStack = httpStack;
        mPool = pool;
    }
    ....
}
```
我们来看看构造方法，看看这个参数：

```
private static int SLOW_REQUEST_THRESHOLD_MS = 3000;
```
从名字看出，最慢的请求阈值（毫秒），也就是超时时间！

那我们再看看这两个：

```
    private static int DEFAULT_POOL_SIZE = 4096;

    protected final ByteArrayPool mPool;
```
这里用了一个Byte数组的缓存池，以及它的大小。
我们来看看这个类的注释：

>  * ByteArrayPool is a source and repository of <code>byte[]</code> objects. Its purpose is to supply those buffers to consumers who need to use them for a short period of time  and then dispose of them. Simply creating and disposing such buffers in the conventional manner canconsiderable heap churn and garbage collection delays on Android, which lacks good management of short-lived heap objects.
>  * It may be advantageous to trade off some memory in the form of a  permanently allocated pool of buffers in order to gain heap performance improvements; that is what this class does.

可以看出，这个是为了byte[]复用的，这样对内存能够起到缓解压力的作用（*in order to gain heap performance improvements*。

因为能力有限，就不擅自分析这个类的实现了，大家可以参照**Trinea前辈的4.2.16节ByteArrayPool.java分析**看看。

OK，这个BasicNetwork中具体实现的performRequest方法，我们先等一下，放到后面再讲~

回到我们的Volley类的newRequestQueue方法中来，我们继续往下分析：

```
        RequestQueue queue;
       
        if (maxDiskCacheBytes <= -1) {
        	// No maximum size specified
        	queue = new RequestQueue(new DiskBasedCache(cacheDir), network);
        }
        else {
        	// Disk cache size specified
        	queue = new RequestQueue(new DiskBasedCache(cacheDir, maxDiskCacheBytes), network);
        }
```
这个开始创建我们的RequestQueue队列。 我们看看DiskBasedCache这个类是什么。
我们看看他的注释：

> Cache implementation that caches files directly onto the hard disk in the specified  directory. The default disk usage size is 5MB, but is configurable.

可以看出，这是用来缓存网络返回的结果的类。它可以自定义缓存的大小，默认是5MB。

结合最后一段代码
```
queue.start();
```
我们来看看RequestQueue这个类：

首先，我们看看它的构造方法：

```
	    /** Used for generating monotonically-increasing sequence numbers for requests. */
    private AtomicInteger mSequenceGenerator = new AtomicInteger();

    /**
     * Staging area for requests that already have a duplicate request in flight.
     *
     * <ul>
     *     <li>containsKey(cacheKey) indicates that there is a request in flight for the given cache
     *          key.</li>
     *     <li>get(cacheKey) returns waiting requests for the given cache key. The in flight request
     *          is <em>not</em> contained in that list. Is null if no requests are staged.</li>
     * </ul>
     */
    private final Map<String, Queue<Request<?>>> mWaitingRequests =
            new HashMap<String, Queue<Request<?>>>();

    /**
     * The set of all requests currently being processed by this RequestQueue. A Request
     * will be in this set if it is waiting in any queue or currently being processed by
     * any dispatcher.
     */
    private final Set<Request<?>> mCurrentRequests = new HashSet<Request<?>>();

    /** The cache triage queue. */
    private final PriorityBlockingQueue<Request<?>> mCacheQueue =
        new PriorityBlockingQueue<Request<?>>();

    /** The queue of requests that are actually going out to the network. */
    private final PriorityBlockingQueue<Request<?>> mNetworkQueue =
        new PriorityBlockingQueue<Request<?>>();

    /** Number of network request dispatcher threads to start. */
    private static final int DEFAULT_NETWORK_THREAD_POOL_SIZE = 4;

    /** Cache interface for retrieving and storing responses. */
    private final Cache mCache;

    /** Network interface for performing requests. */
    private final Network mNetwork;

    /** Response delivery mechanism. */
    private final ResponseDelivery mDelivery;

    /** The network dispatchers. */
    private NetworkDispatcher[] mDispatchers;

    /** The cache dispatcher. */
    private CacheDispatcher mCacheDispatcher;

    private List<RequestFinishedListener> mFinishedListeners =
            new ArrayList<RequestFinishedListener>();
            
    /**
     * Creates the worker pool. Processing will not begin until {@link #start()} is called.
     *
     * @param cache A Cache to use for persisting responses to disk
     * @param network A Network interface for performing HTTP requests
     * @param threadPoolSize Number of network dispatcher threads to create
     * @param delivery A ResponseDelivery interface for posting responses and errors
     */
    public RequestQueue(Cache cache, Network network, int threadPoolSize,
            ResponseDelivery delivery) {
        mCache = cache;
        mNetwork = network;
        mDispatchers = new NetworkDispatcher[threadPoolSize];
        mDelivery = delivery;
    }

    public RequestQueue(Cache cache, Network network, int threadPoolSize) {
        this(cache, network, threadPoolSize,
                new ExecutorDelivery(new Handler(Looper.getMainLooper())));
    }

    public RequestQueue(Cache cache, Network network) {
        this(cache, network, DEFAULT_NETWORK_THREAD_POOL_SIZE);
    }
```
我们可以这里创建了一个
`new ExecutorDelivery(new Handler(Looper.getMainLooper()))`
这个，我们看看这个类的注释

> Delivers responses and errors.

可以知道，它是用来分发传输网络请求结果和错误的。

我们继续往上看RequestQueue类的构造方法中

```
mDispatchers = new NetworkDispatcher[threadPoolSize];
```
这里创建了一个默认大小为4的 网络分发调度线程。

> Trinea前辈认为这里是可以根据 当前CPU数量来动态分配的，或者网络请求类型来优化数量。

那么我们现在可以来具体分析queue.start()的start方法了。

具体代码：
```
    /**
     * Starts the dispatchers in this queue.
     */
    public void start() {
        stop();  // Make sure any currently running dispatchers are stopped.
        // Create the cache dispatcher and start it.
        mCacheDispatcher = new CacheDispatcher(mCacheQueue, mNetworkQueue, mCache, mDelivery);
        mCacheDispatcher.start();

        // Create network dispatchers (and corresponding threads) up to the pool size.
        for (int i = 0; i < mDispatchers.length; i++) {
            NetworkDispatcher networkDispatcher = new NetworkDispatcher(mNetworkQueue, mNetwork,
                    mCache, mDelivery);
            mDispatchers[i] = networkDispatcher;
            networkDispatcher.start();
        }
    }
```
首先它创建了一个缓存的调度线程CacheDispatcher，并启动。
创建了四个网络任务调度线程NetworkDispatcher，并启动。

那我们现在先小结一下这个RequestQueue类里面有什么。

 1. 一个分发结果的ResponseDelivery类：mDelivery
 2. 一个负责缓存的CacheDispatcher类：mCacheDispatcher
 3. 四个负责网络任务分发的NetworkDispatcher类（数组形式）：mDispatchers
 4. 网络请求的客户端Network：mNetwork
 5. 一个具体缓存的实例Cache：mCache
 6. 一个带有优先级的缓存任务队列：mCacheQueue
 7. 一个带有优先级的网络任务队列：mNetworkQueue
 8. 一个当前正在执行任务的Set集合：mCurrentRequests
 9. 其他几个....
 
 
<h1>3.结束语</h1>
至此，我们就分析完了队列的创建，都做了些什么事情了！

看起来类很多，但是结构还是很清晰的。 希望大家能够看完这篇文章之后再自己照着源码看一篇。

下一篇文章我们将来重点看看Volley的核心，两个调度线程类：
CacheDispatcher、NetworkDispatcher以及任务分派的：ResponseDelivery。

