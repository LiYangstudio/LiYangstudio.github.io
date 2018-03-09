

<!-- more -->
## MemoryCache

**接口类**

定义了五个基本方法：

```

/**
 * Puts value into cache by key
 *
 * @return <b>true</b> - if value was put into cache successfully, <b>false</b> - if value was <b>not</b> put into
 * cache
 */
boolean put(String key, Bitmap value);

/** Returns value by key. If there is no value for key then null will be returned. */
Bitmap get(String key);

/** Removes item by key */
Bitmap remove(String key);

/** Returns all keys of cache */
Collection<String> keys();

/** Remove all items from cache */
void clear();
```



## BaseMemoryCache

**抽象类**

> Base memory cache. Implements common functionality for memory cache. Provides object references ( * {@linkplain Reference not strong}) storing.



* 基本类. 实现了内存缓存的基本函数，可以指定对象的引用类型

* 实现了`MemoryCache`接口的所有方法

* 使用了`softMap = Collections.synchronizedMap(new HashMap<String, Reference<Bitmap>>());`实现图片保存（可以看到这有一个Reference）



### 源码分析

#### 1.put方法

```

@Override

public boolean put(String key, Bitmap value)  {

    softMap.put(key, createReference(value));

    return true;

}



/** Creates {@linkplain Reference not strong} reference of value */

protected abstract Reference<Bitmap> createReference(Bitmap value);

```

过程：

* 调用`put`方法的时候会调用`createReference`

* 指定一个抽象方法 用来子类决定使用什么类型的引用



#### 2.获取keys

```

@Override

public Collection<String> keys() {

    synchronized (softMap) {

    return new HashSet<String>(softMap.keySet());

    }

}

```

* 获取键返回的是一个包装之后的set集合



## LimitedMemoryCache

**抽象类 、 继承自`BaseMemoryCache`**

```

/**

* Limited cache. Provides object storing. Size of all stored bitmaps will not to exceed size limit (

* {@link #getSizeLimit()}).<br />

* <br />

* <b>NOTE:</b> This cache uses strong and weak references for stored Bitmaps. Strong references - for limited count of

* Bitmaps (depends on cache size), weak references - for all other cached Bitmaps.

*

* @author Sergey Tarasevich (nostra13[at]gmail[dot]com)

* @see BaseMemoryCache

* @since 1.0.0

*/

```

* 有限制缓存大小，来限定`getSizeLimite()`

* 使用强、软引用进行Bitmap的缓存

    * 强引用：强引用保存Bitmap（也取决于缓存的大小）

    * 弱引用：所有的缓存的Bitmap



### 源码分析

#### 1.构造函数与成员变量

```

private static final int MAX_NORMAL_CACHE_SIZE_IN_MB = 16;

private static final int MAX_NORMAL_CACHE_SIZE = MAX_NORMAL_CHACHE_SIZE_IN_MB * 1024 * 1024;//最大缓存 是16MB

private final AtomicInteger cacheSize;

/**

* Contains strong references to stored objects. Each next object is added last. If hard cache size will exceed

* limit then first object is deleted (but it continue exist at {@link #softMap} and can be collected by GC at any

* time)

*/

private final List<Bitmap> hardCache = Collections.synchronizedList(new LinkedList<Bitmap>());



public LimitedMemoryCache (int sizeLimit) {

    //...

    cacheSize = new AtomicInteger();

    if (size > MAX_NORMAL_CACHE_SIZE) {

        L.w("You set too large memory cache size (more than %d Mb)", MAX_NORMAL_CACHE_SIZE);    

    }

}





```

* 设置了16MB为推荐的最大缓存大小 可能是因为原生系统一个应用分配的大小是16MB的缘故

* 使用`AtomicInteger`原子类型的整型，作为计算当前缓存图片的总大小（用与移除图片）

* 在`BaseMemoryCache.softMap`的基础上，增加了一个`hardMap`。



注意：

**`hardMap`有一段注释：如果强引用集合（hardMap）的大小超过了限定大小，那么第一个对象将会被从这个集合中移除；但是它依旧会存在`softMap`中，而且这样并不妨碍GC**



个人见解：

**应该是UIL的多级缓存，在缓存满了之后且不妨碍GC的情况下，又尽可能缓存多的图片。**

#### 2.put方法

```

@Override
public boolean put(String key, Bitmap value) {
   boolean putSuccessfully = false;
   // Try to add value to hard cache
   int valueSize = getSize(value);
   int sizeLimit = getSizeLimit();
   int curCacheSize = cacheSize.get();
   if (valueSize < sizeLimit) {
      while (curCacheSize + valueSize > sizeLimit) {
         Bitmap removedValue = removeNext();
         if (hardCache.remove(removedValue)) {
            curCacheSize = cacheSize.addAndGet(-getSize(removedValue));
         }
      }
      hardCache.add(value);
      cacheSize.addAndGet(valueSize);

      putSuccessfully = true;
   }
   // Add value to soft cache
   super.put(key, value);
   return putSuccessfully;
}


protected abstract Bitmap removeNext();
```

* `removeNext()` 超过规定大小之后会调用这个方法来移除。 这里是为了扩展性考虑，从源码的impl包可以看出，作者实现了多种数据结构的`MemoryCache`。

* 超过缓存大小，只从`hardMap`中移除的，没有`softMap`的；也印证了上面的注释。

* 这里的方法都没使用同步锁。直接使用并发HashMap，以及使用了原子整型`AtomicInteger`进行缓存大小的计算。



## 总结

* `MemoryCache`定义接口方法

* `BaseMemoryCache`使用**可指定引用类型的并发HashMap**`softMap`，用于最下级缓存。

* `LimitedMemoryCache`使用**强引用类型的并发HashMap**`hardMap`以及指定最大缓存大小，用于第二级缓存。 并且在超过缓存大小的时候，会从`hardMap`释放引用，而不从`softMap`释放。


