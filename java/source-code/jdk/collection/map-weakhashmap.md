# `WeakHashMap<K,V>`

## 设计

链地址法，不转红黑树。

使用 `WeakReference` 做 `Entry` 利用 GC 来回收条目

## 扩容策略

```java
/**
* The default initial capacity -- MUST be a power of two.
*/
private static final int DEFAULT_INITIAL_CAPACITY = 16;

/**
* The load factor used when none specified in constructor.
*/
private static final float DEFAULT_LOAD_FACTOR = 0.75f;
```

达到阈值直接扩容为原来两倍

```java
if (++size >= threshold)
    resize(tab.length * 2);
```

## 开源组件拓展

### Tomcat

```java

public final class ConcurrentCache<K,V> {

    private final int size;

    private final Map<K,V> eden;

    private final Map<K,V> longterm;

    public ConcurrentCache(int size) {
        this.size = size;
        this.eden = new ConcurrentHashMap<>(size);
        this.longterm = new WeakHashMap<>(size);
    }

    public V get(K k) {
        V v = this.eden.get(k);
        if (v == null) {
            synchronized (longterm) {
                v = this.longterm.get(k);
            }
            if (v != null) {
                this.eden.put(k, v);
            }
        }
        return v;
    }

    public void put(K k, V v) {
        if (this.eden.size() >= size) {
            synchronized (longterm) {
                this.longterm.putAll(this.eden);
            }
            this.eden.clear();
        }
        this.eden.put(k, v);
    }
}
```

## 参考

- [Java Reference - TODO](../lang/ref/reference.md)
- [WeakHashMap 的实际使用场景有哪些？ - 知乎](https://www.zhihu.com/question/46648593)
- [ConcurrentCache - Tomcat](https://github.com/apache/tomcat/blob/trunk/java/org/apache/tomcat/util/collections/ConcurrentCache.java)
