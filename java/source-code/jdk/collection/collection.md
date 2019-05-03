# Collection 接口

[Java™  8 - Collection][1]。

本接口的 Demo 可在 [furry-octo-doodle](https://github.com/c-rainstorm/furry-octo-doodle) 仓库的 `me.rainstorm.fod.CollectionTest` 中找到。`CollectionTest` 是集合框架测试类的基类，后续所有具体实现类的测试类都继承该类来获取通用的测试用例

本文中的JDK源码是 `AbstractCollection` 类提供的通用实现。

---

- [Collection 接口](#collection-%E6%8E%A5%E5%8F%A3)
  - [[Java 集合框架概览][2]](#java-%E9%9B%86%E5%90%88%E6%A1%86%E6%9E%B6%E6%A6%82%E8%A7%882)
    - [通用实现](#%E9%80%9A%E7%94%A8%E5%AE%9E%E7%8E%B0)
    - [并发实现](#%E5%B9%B6%E5%8F%91%E5%AE%9E%E7%8E%B0)
  - [`add(E e)`](#adde-e)
  - [`addAll(Collection<? extends E> c)`](#addallcollection-extends-e-c)
    - [源码实现](#%E6%BA%90%E7%A0%81%E5%AE%9E%E7%8E%B0)
  - [`clear()`](#clear)
    - [源码实现](#%E6%BA%90%E7%A0%81%E5%AE%9E%E7%8E%B0-1)
  - [`contains(Object o)`](#containsobject-o)
  - [`equals(Object o)`](#equalsobject-o)
  - [`isEmpty()`](#isempty)
  - [`parallelStream()`](#parallelstream)
  - [`remove(Object o)`](#removeobject-o)
  - [`removeIf(Predicate<? super E> filter)`](#removeifpredicate-super-e-filter)
    - [源码实现](#%E6%BA%90%E7%A0%81%E5%AE%9E%E7%8E%B0-2)
  - [`stream()`](#stream)
  - [`toArray()`](#toarray)
    - [源码实现](#%E6%BA%90%E7%A0%81%E5%AE%9E%E7%8E%B0-3)

---

## [Java 集合框架概览][2]

这个接口是 Java 集合框架体系结构的根节点，这个接口没有直接的实现类，JDK 提供了更专用的子接口，Set、List、Map 等。

该接口所有的通用实现一般都至少提供两个构造函数，一个是无参构造函数，一个是只有 `Collection` 参数的构造函数，这使得集合框架中的任意实现类之间可以方便的进行转换。

不同的实现类对 `null` 元素的支持不同，如果不支持，会抛出 `NullPointerException`。针对每个实现类，我们后续单独分析。

该接口里很多方法都依赖 `equals()` 方法。比如 `contains()` 等。

底层需要递归遍历的方法，如果集合自身是集合的一个元素，会出现异常。比如 `clone()`、`equals()`、`hashCode()`、`toString()`。以 `toString()` 为例，一般集合调用整个方法的时候，会遍历整个集合，调用集合内每个元素的 `toString`， 如果集合自身是集合的一个元素。那么递归调用 `toString()` 方法会一直无法拿到结果，除非 OOM 或者有其他异常出现。

### 通用实现

// TODO 每学习一个，将其归类到下面的表格中，将其标注出来方便查看学习进度。

|接口| 哈希表 |可变长数组|平衡树|链表|哈希表+链表|
|:---:|:---:|:---:|:---:|:---:|:---:|
|Set|HashSet||TreeSet||LinkedhashSet|
|List||ArrayList||LinkedList||
|Deque||ArrayDeque||LinkedList||
|Map|HashMap||TreeMap||LinkedhashMap|

### 并发实现

// TODO 每学习一个，将其归类到下面的表格中

|接口| 哈希表 |可变长数组|平衡树|链表|哈希表+链表|
|:---:|:---:|:---:|:---:|:---:|:---:|
|Set||||||
|List||||||
|Deque||||||
|Map||||||

## `add(E e)`

添加元素到集合

添加重复元素

```java
@Test
public void addDup() {
    assert !collection.isEmpty();

    int originSize = collection.size();
    String firstElement = collection.stream().findFirst().get();
    collection.add(firstElement);

    if (canContainDup()) {
        assert collection.size() == originSize + 1;
    } else {
        assert collection.size() == originSize;
    }
}

```

添加空值

```java
@Test
public void addNull() {
    assert collection != null;

    Exception exception = null;
    int originSize = collection.size();
    try {
        collection.add(null);
        LOGGER.info("add null success");
    } catch (Exception e) {
        exception = e;
        LOGGER.error("add null failure" + e.getMessage());
    }

    if (canContainNull()) {
        assert exception == null;
        assert collection.size() == originSize + 1;
    } else {
        assert exception != null;
        assert collection.size() == originSize;
    }
}
```

## `addAll(Collection<? extends E> c)`

批量添加元素

```java
@Test
public void addAll() {
    assert collection != null;

    Collection<String> newUUIDs = initWithRandomElement();
    assert newUUIDs != null && newUUIDs.size() == DEFAULT_INIT_SIZE;

    int originSize = collection.size();
    collection.addAll(newUUIDs);

    assert !canContainDup() || collection.size() == originSize + newUUIDs.size();
}

private Collection<String> initWithRandomElement() {
    // 每个具体实现类必须复写 createEmptyCollection 来创建默认容量的空集合
    Collection<String> collection = createEmptyCollection(DEFAULT_INIT_SIZE);
    // 使用 size() 而不是用 for + 下标的方式，是因为 set 会有去重操作，
    // size() 可以保证不管是哪种具体实现，到最后都会有 DEFAULT_INIT_SIZE 个元素
    while (collection.size() < DEFAULT_INIT_SIZE) {
        collection.add(UUID.randomUUID().toString());
    }

    return collection;
}
```

### 源码实现

```java
public boolean addAll(Collection<? extends E> c) {
    boolean modified = false;
    // 遍历被添加的集合，将每个元素逐一加入
    // 如果 c 为空，直接会抛出空指针；
    for (E e : c)
        // 如果有一个加入成功则结果为 true
        // TIP：如果 c 可以包含 null，当前集合不允许添加 null 会抛出空指针
        // 在不允许 null 元素的实现调用 addAll 要谨慎
        if (add(e))
            modified = true;
    return modified;
}
```

## `clear()`

清空集合元素

### 源码实现

因为调用了迭代器的 remove 方法，具体实现类的 clear 性能有所不同，需要看具体的实现。

```java
public void clear() {
    Iterator<E> it = iterator();
    while (it.hasNext()) {
        it.next();
        it.remove();
    }
}
```

## `contains(Object o)`

默认实现遍历集合，调用 equals() 方法判断是否相等

## `equals(Object o)`


## `isEmpty()`

```java
public boolean isEmpty() {
    return size() == 0;
}
```

## `parallelStream()`

`@since 1.8`

// TODO 深入 `Spliterator` 接口

默认实现使用 `spliterator` 生成了一个并行流。

```java
default Stream<E> parallelStream() {
    return StreamSupport.stream(spliterator(), true);
}
```

## `remove(Object o)`

默认使用迭代器遍历，找到 equals 的元素用迭代器的 remove 方法删除。

## `removeIf(Predicate<? super E> filter)`

`@since 1.8`

//TODO java 8 函数式编程

删除满足条件的元素

```java
@Test
public void removeIf() {
    assert collection != null;

    // 定义一个谓词，首字母是数字
    Predicate<String> filter = (x) -> x != null && Character.isDigit(x.charAt(0));

    // 原始元素个数
    int originSize = collection.size();
    // 满足条件的元素总数
    int filterSatisfyNum = (int) collection.stream().filter(filter).count();
    // 删除满足条件的元素
    collection.removeIf(filter);

    // 2019-04-27 23:05:05,891 [main] DEBUG me.rainstorm.fod.CollectionTest - before: 10, filterSatisfyNum: 6, after: 4
    LOGGER.debug(String.format("before: %s, filterSatisfyNum: %s, after: %s", originSize, filterSatisfyNum, collection.size()));
    assert collection.size() == originSize - filterSatisfyNum;
}
```

### 源码实现

```java
default boolean removeIf(Predicate<? super E> filter) {
    // filter 为空抛空指针
    Objects.requireNonNull(filter);
    boolean removed = false;
    final Iterator<E> each = iterator();
    while (each.hasNext()) {
        // 迭代器遍历，满足条件就删除
        if (filter.test(each.next())) {
            each.remove();
            removed = true;
        }
    }
    return removed;
}
```

## `stream()`

## `toArray()`

方法保证返回对象数组的长度等于集合内元素数

### 源码实现

```java
public Object[] toArray() {
    // Estimate size of array; be prepared to see more or fewer elements
    // 如果出现这种容量变动的情况一定是调用 迭代器的 remove 方法删除了元素
    Object[] r = new Object[size()];
    Iterator<E> it = iterator();
    for (int i = 0; i < r.length; i++) {
        if (! it.hasNext()) // fewer elements than expected
            return Arrays.copyOf(r, i);
        r[i] = it.next();
    }
    return it.hasNext() ? finishToArray(r, it) : r;
}

private static <T> T[] finishToArray(T[] r, Iterator<?> it) {
    int i = r.length;
    while (it.hasNext()) {
        int cap = r.length;
        if (i == cap) {
            // 扩容操作原容量的 3/2 左右
            int newCap = cap + (cap >> 1) + 1;
            // overflow-conscious code
            if (newCap - MAX_ARRAY_SIZE > 0)
                newCap = hugeCapacity(cap + 1);
            r = Arrays.copyOf(r, newCap);
        }
        r[i++] = (T)it.next();
    }
    // trim if overallocated
    // 多分配了就重新复制到新数组，保证元素数与数组长度一致
    return (i == r.length) ? r : Arrays.copyOf(r, i);
}
```

[1]:https://docs.oracle.com/javase/8/docs/api/java/util/Collection.html "Java™  8 - Collection<E>"
[2]:https://docs.oracle.com/javase/8/docs/technotes/guides/collections/overview.html "Collections Framework Overview"
