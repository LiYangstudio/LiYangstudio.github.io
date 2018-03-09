

<!-- more -->
## LRULimitedMemoryCache

**继承自LimitedMemoryCache**

```

/**
 * Limited {@link Bitmap bitmap} cache.
 * Provides {@link Bitmap bitmaps} storing.
 * Size of all stored bitmaps will not to exceed size limit.
 * When cache reaches limit size
 * then the least recently used bitmap is deleted from cache.<br />
 * <br />
 * <b>NOTE:</b>
 * This cache uses strong and weak references for stored Bitmaps.
 * Strong references - for limited count of
 * Bitmaps (depends on cache size),
 * weak references - for all other cached Bitmaps.
 *
 * @author Sergey Tarasevich (nostra13[at]gmail[dot]com)
 * @since 1.3.0
 */
```

* 当达到缓存限制值的时候，最近没有被使用的Bitmap会从缓存中移除

* 使用强引用保存Bitmap（取决于缓存大小）

* 使用弱引用保存所有的Bitmap （这个特性是继承自LimitedMemoryCache）



### 源码分析

```

public class LRULimitedMemoryCache extends LimitedMemoryCache {

   private static final int INITIAL_CAPACITY = 10;
   private static final float LOAD_FACTOR = 1.1f;

   /** Cache providing Least-Recently-Used logic */
   private final Map<String, Bitmap> lruCache =
         Collections.synchronizedMap(
               new LinkedHashMap<String, Bitmap>(INITIAL_CAPACITY, LOAD_FACTOR, true)
         );

   /** @param maxSize Maximum sum of the sizes of the Bitmaps in this cache */
   public LRULimitedMemoryCache(int maxSize) {
      super(maxSize);
   }

   @Override
   public boolean put(String key, Bitmap value) {
      if (super.put(key, value)) {
         lruCache.put(key, value);
         return true;
      } else {
         return false;
      }
   }

   @Override
   public Bitmap get(String key) {
      lruCache.get(key); // call "get" for LRU logic
      return super.get(key);
   }

   @Override
   public Bitmap remove(String key) {
      lruCache.remove(key);
      return super.remove(key);
   }

   @Override
   public void clear() {
      lruCache.clear();
      super.clear();
   }

   @Override
   protected int getSize(Bitmap value) {
      return value.getRowBytes() * value.getHeight();
   }

   @Override
   protected Bitmap removeNext() {
      Bitmap mostLongUsedValue = null;
      synchronized (lruCache) {
         Iterator<Entry<String, Bitmap>> it = lruCache.entrySet().iterator();
         if (it.hasNext()) {
            Entry<String, Bitmap> entry = it.next();
            mostLongUsedValue = entry.getValue();
            it.remove();
         }
      }
      return mostLongUsedValue;
   }

   @Override
   protected Reference<Bitmap> createReference(Bitmap value) {
      return new WeakReference<Bitmap>(value);
   }
}
```

* 使用`LinkedHashMap`的特性来实现LRU（和LruCache原理相同）

* `add`、`remove`等操作都会调用super

* `softMap`指定的是`WeakReference`

* `removeNext`是根据`LRU`特性，直接移除第一个



### 小结

* 继承自`LimitedMemoryCache` 所以实现了**两层的内存缓存**



## LRUMemoryCache

**直接实现MemoryCache接口**

```

/**
 * A cache that holds strong references
 * to a limited number of Bitmaps.
 * Each time a Bitmap is accessed, it is moved to
 * the head of a queue.
 * When a Bitmap is added to a full cache,
 * the Bitmap at the end of that queue is evicted and may
 * become eligible for garbage collection.<br />
 * <br />
 * <b>NOTE:</b>
 * This cache uses only strong references for stored Bitmaps.
 *
 * @author Sergey Tarasevich (nostra13[at]gmail[dot]com)
 * @since 1.8.1
 */
```

* 每次被获取的`Bitmap`都会被放到队列的头部

* 当缓存大小满了之后，处于队尾的将被移除

* 这个只用了强缓存（因为直接实现的`MemoryCache`没有继承于`LimitedMemoryCache`）



### 源码分析

* 源代码基本和官方`LruCache`一样



## 总结

* `LRULimitedMemoryCache`继承于`BaseMemoryCache`实现了两级的内存缓存（`softMap`和`hardMap`）

* `LRUMemoryCache`直接实现`MemoryCache` 值实现了强引用缓存

* 两者都具有并发的能力. 在实现细节上稍有不同。

    * `LRULimitedMemoryCache`基于`LimitedMemoryCache`使用`并发HashMap`和`AtomicInteger`，方法内不加同步代码块。

    * `LRUMemoryCache`使用`synchronized`关键字在方法中实现同步

* `LRUMemoryCache`和官方的`LruCache`基本一致


