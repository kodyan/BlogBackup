title: Android中的LruCache
date: 2016/4/21 20:45:37
categories: 
tags: Android
---
LRU是Least Recently Used的缩写，即“最近最少使用”，说明LRU缓存算法的淘汰策略是把最近最少使用的数据移除，让出内存给最新读取的数据。下面看一下Android中的LruCache。

<!--more-->

## android.util.LruCache

这个LruCache在android.util包下，是API level 12引入的，对于API level 12之前的系统可以使用support library中的LruCache。先来看看android.util.LruCache的源码。

首先是成员变量：

```java
private final LinkedHashMap<K, V> map;

    /** Size of this cache in units. Not necessarily the number of elements. */
    private int size;
    private int maxSize;

    private int putCount;
    private int createCount;
    private int evictionCount;
    private int hitCount;
    private int missCount;
```
LruCache内部使用一个LinkedHashMap作为存储容器，并对各种操作进行计次。

构造器：

```java
public LruCache(int maxSize) {
        if (maxSize <= 0) {
            throw new IllegalArgumentException("maxSize <= 0");
        }
        this.maxSize = maxSize;
        this.map = new LinkedHashMap<K, V>(0, 0.75f, true);
    }
```
构造器的参数maxSize用于指定缓存的最大容量，并初始化一个LinkedHashMap，顺便看看这个LinkedHashMap的构造函数：

```java
public LinkedHashMap(int initialCapacity, float loadFactor, boolean accessOrder) {
        super(initialCapacity, loadFactor);
        init();
        this.accessOrder = accessOrder;
    }
```
initialCapacity即初始容量设为0，装填因子loadFactor设为0.75，accessOrder设为true，即链表中的元素按照最近最少访问到最多访问排序。这里设置的装填因子为0.75，设置其它值行不行呢？在LinkedHashMap这个构造器中只是将loadFactor作为参数传给了父类构造器，该父类构造器如下：

```java
public HashMap(int capacity, float loadFactor) {
        this(capacity);

        if (loadFactor <= 0 || Float.isNaN(loadFactor)) {
            throw new IllegalArgumentException("Load factor: " + loadFactor);
        }

        /*
         * Note that this implementation ignores loadFactor; it always uses
         * a load factor of 3/4. This simplifies the code and generally
         * improves performance.
         */
    }
```
调用了HashMap的构造器，可以看到只是对loadFactor进行了合法检查，除此之外没有其他调用或赋值操作，Note中解释了，这个loadFactor没用，装填因子永远使用3/4，也就是0.75。所以在构造LinkedHashMap时，设了装填因子也没什么用。

继续看LruCache，resize方法更新链表容量，调用trimToSize方法。

```java
public void resize(int maxSize) {
        if (maxSize <= 0) {
            throw new IllegalArgumentException("maxSize <= 0");
        }

        synchronized (this) {
            this.maxSize = maxSize;
        }
        trimToSize(maxSize);
    }
```
先看get方法，key为空会抛异常，取出对应的value，若不为空则命中次数hitCount加1并return这个value，否则missCount加1。该value为空时继续向下执行，根据key尝试创建value，如果创建返回的createdValue是null，那就确实没有该值，若创建操作返回的createdValue不为null，则尝试把createdValue放回map，若存在旧值则返回旧值，否则返回这个createdValue。

```java
public final V get(K key) {
        if (key == null) {
            throw new NullPointerException("key == null");
        }

        V mapValue;
        synchronized (this) {
            mapValue = map.get(key);
            if (mapValue != null) {
                hitCount++;
                return mapValue;
            }
            missCount++;
        }

        V createdValue = create(key);
        if (createdValue == null) {
            return null;
        }

        synchronized (this) {
            createCount++;
            mapValue = map.put(key, createdValue);

            if (mapValue != null) {
                // There was a conflict so undo that last put
                map.put(key, mapValue);
            } else {
                size += safeSizeOf(key, createdValue);
            }
        }

        if (mapValue != null) {
            entryRemoved(false, key, createdValue, mapValue);
            return mapValue;
        } else {
            trimToSize(maxSize);
            return createdValue;
        }
    }
```

put方法将键值对放入map，重新计算大小之后调用trimToSize方法，删除访问次数最少的元素。

```java
 public final V put(K key, V value) {
        if (key == null || value == null) {
            throw new NullPointerException("key == null || value == null");
        }

        V previous;
        synchronized (this) {
            putCount++;
            size += safeSizeOf(key, value);
            previous = map.put(key, value);
            if (previous != null) {
                size -= safeSizeOf(key, previous);
            }
        }

        if (previous != null) {
            entryRemoved(false, key, previous, value);
        }

        trimToSize(maxSize);
        return previous;
    }
```

trimToSize方法中会一直尝试删除队首元素即访问次数最少的元素，直到size不超过最大容量。

```java
public void trimToSize(int maxSize) {
        while (true) {
            K key;
            V value;
            synchronized (this) {
                if (size < 0 || (map.isEmpty() && size != 0)) {
                    throw new IllegalStateException(getClass().getName()
                            + ".sizeOf() is reporting inconsistent results!");
                }

                if (size <= maxSize) {
                    break;
                }

                Map.Entry<K, V> toEvict = map.eldest();
                if (toEvict == null) {
                    break;
                }

                key = toEvict.getKey();
                value = toEvict.getValue();
                map.remove(key);
                size -= safeSizeOf(key, value);
                evictionCount++;
            }

            entryRemoved(true, key, value, null);
        }
    }
```

## android.support.v4.util.LruCache
support v4包中的LruCache可以用于API level 12之前的系统，和android.util包的LruCache的区别是在trimToSize中获取将要删除元素的方法不一样：

- android.util.LruCache 

```java
Map.Entry<K, V> toEvict = map.eldest();
```
- android.support.v4.util.LruCache

```java
Map.Entry<K, V> toEvict = map.entrySet().iterator().next();
```
LinkedHashMap的eldest()方法已经被标注为@hide，所以使用android.support.v4.util.LruCache更加保险一点。