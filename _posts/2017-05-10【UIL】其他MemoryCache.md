

<!-- more -->
## FuzzyKeyMemory

```

/**
 * Decorator for {@link MemoryCache}.
 * Provides special feature for cache:
 * some different keys are considered as
 * equals (using {@link Comparator comparator}).
 * And when you try to put some value into cache by key so entries with
 * "equals" keys will be removed from cache before.<br />
 * <b>NOTE:</b>
 * Used for internal needs. Normally you don't need to use this class.
 *
 * @author Sergey Tarasevich (nostra13[at]gmail[dot]com)
 * @since 1.0.0
 */
```

* MemoryCache的装饰类

* 用于当你加入某个键和值时候，移除另外一个键值

* 具体实现通过`Comparator`

* 用在内部，一般情况下开发者用不到



### 源代码分析

```

public class FuzzyKeyMemoryCache implements MemoryCache {

   private final MemoryCache cache;
   private final Comparator<String> keyComparator;

   public FuzzyKeyMemoryCache(MemoryCache cache, Comparator<String> keyComparator) {
      this.cache = cache;
      this.keyComparator = keyComparator;
   }

   @Override
   public boolean put(String key, Bitmap value) {
      // Search equal key and remove this entry
      synchronized (cache) {
         String keyToRemove = null;
         for (String cacheKey : cache.keys()) {//遍历旧键
            if (keyComparator.compare(key, cacheKey) == 0) {//通过compare对比一致
               keyToRemove = cacheKey;
               break;
            }
         }
         if (keyToRemove != null) {
            cache.remove(keyToRemove);
         }
      }
      return cache.put(key, value);
   }

//....
}
```

* 传入被包装的`MemoryCache`和键比较器`Comparetor`

* `put`的时候会遍历旧键，与`Comparetor`对比。如果是需替换的，则移除键值；然后加入传入的键值到缓存中



## UsingFreqLimitedMemoryCache

```

/**
 * Limited {@link Bitmap bitmap} cache.
 * Provides {@link Bitmap bitmaps} storing.
 * Size of all stored bitmaps will not to
 * exceed size limit.
 * When cache reaches limit size then the bitmap
 * which used the least frequently is deleted from
 * cache.<br />
 * <br />
 * <b>NOTE:</b> This cache uses strong and
 * weak references for stored Bitmaps.
 * Strong references - for limited count of
 * Bitmaps (depends on cache size),
 * weak references - for all other cached Bitmaps.
 *
 * @author Sergey Tarasevich (nostra13[at]gmail[dot]com)
 * @since 1.0.0
 */
```

* 与LRU直接相反；在缓存满的时候，使用最多次的会被移除



### 源代码分析

```

public class UsingFreqLimitedMemoryCache extends LimitedMemoryCache {
   /**
    * Contains strong references to stored objects (keys) and last object usage date (in milliseconds). If hard cache
    * size will exceed limit then object with the least frequently usage is deleted (but it continue exist at
    * {@link #softMap} and can be collected by GC at any time)
    */
   private final Map<Bitmap, Integer> usingCounts = Collections.synchronizedMap(new HashMap<Bitmap, Integer>());

   public UsingFreqLimitedMemoryCache(int sizeLimit) {
      super(sizeLimit);
   }

   @Override
   public boolean put(String key, Bitmap value) {
      if (super.put(key, value)) {
         usingCounts.put(value, 0);
         return true;
      } else {
         return false;
      }
   }

   @Override
   public Bitmap get(String key) {
      Bitmap value = super.get(key);
      // Increment usage count for value if value is contained in hardCahe
      if (value != null) {
         Integer usageCount = usingCounts.get(value);
         if (usageCount != null) {
            usingCounts.put(value, usageCount + 1);
         }
      }
      return value;
   }

 //...

   @Override
   protected Bitmap removeNext() {
      Integer minUsageCount = null;
      Bitmap leastUsedValue = null;
      Set<Entry<Bitmap, Integer>> entries = usingCounts.entrySet();
      synchronized (usingCounts) {
         for (Entry<Bitmap, Integer> entry : entries) {
            if (leastUsedValue == null) {
               leastUsedValue = entry.getKey();
               minUsageCount = entry.getValue();
            } else {
               Integer lastValueUsage = entry.getValue();
               if (lastValueUsage < minUsageCount) {
                  minUsageCount = lastValueUsage;
                  leastUsedValue = entry.getKey();
               }
            }
         }
      }
      usingCounts.remove(leastUsedValue);
      return leastUsedValue;
   }

   @Override
   protected Reference<Bitmap> createReference(Bitmap value) {
      return new WeakReference<Bitmap>(value);
   }
}


```

* 使用`HashMap`键为`Bitmap`值为使用次数

* 重写`removeNext`找到次数最多的直接移除



## 其他

* FIFOLimitedMemoryCache

* LargestLimitedMemoryCache

* LimitedAgeMemoryCache

其他的原理都比较简单，留个印象，可能用到的时候再去看就好了


