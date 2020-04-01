# CopyOnWriteArrayList

写时复制底层数组，新数组的引用会直接替换老的数组引用，复制开销较大，适合用于读远远超过写的频率。

线程安全。

## 设计思路

读写分离。

读操作时会先拿到数组的引用，写时复制后写入新数组

```java
public int indexOf(Object o) {
    // 读操作先拿数组引用
    Object[] elements = getArray();
    return indexOf(o, elements, 0, elements.length);
}
```

```java
public boolean add(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        // 写前先复制，并发过来的读请求会使用旧数组
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        newElements[len] = e;
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
}
```

## 迭代器遍历

不支持修改，因为 snapshot 为创建迭代器时的数组引用，这个数组在迭代期间不会被修改，因为写操作时会复制新的数组，所以在遍历时不会有 `ConcurrentModificationException`

```java
static final class COWIterator<E> implements ListIterator<E> {
    /** Snapshot of the array */
    private final Object[] snapshot;
    /** Index of element to be returned by subsequent call to next.  */
    private int cursor;

    /**
    * Not supported. Always throws UnsupportedOperationException.
    * @throws UnsupportedOperationException always; {@code remove}
    *         is not supported by this iterator.
    */
    public void remove() {
        throw new UnsupportedOperationException();
    }

    /**
    * Not supported. Always throws UnsupportedOperationException.
    * @throws UnsupportedOperationException always; {@code set}
    *         is not supported by this iterator.
    */
    public void set(E e) {
        throw new UnsupportedOperationException();
    }

    /**
    * Not supported. Always throws UnsupportedOperationException.
    * @throws UnsupportedOperationException always; {@code add}
    *         is not supported by this iterator.
    */
    public void add(E e) {
        throw new UnsupportedOperationException();
    }
}
```

## 可重入锁保证一致性

```java
public boolean add(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        newElements[len] = e;
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
}
```

## 开源组件拓展

### Apollo

apollo 使用 `CopyOnWriteArrayList` 维护配置变更监听器列表。非常典型的读多写少场景。

// 有个问题，因为这个 list 每次都是添加一个监听器，若监听器比较多，性能有点尴尬。

[Apollo配置中心介绍#36-客户端监听配置变化](https://github.com/ctripcorp/apollo/wiki/Apollo%E9%85%8D%E7%BD%AE%E4%B8%AD%E5%BF%83%E4%BB%8B%E7%BB%8D#36-%E5%AE%A2%E6%88%B7%E7%AB%AF%E7%9B%91%E5%90%AC%E9%85%8D%E7%BD%AE%E5%8F%98%E5%8C%96)

## 最后

- [并发容器之CopyOnWriteArrayList - 你听___](https://juejin.im/post/5aeeb55f5188256715478c21)
