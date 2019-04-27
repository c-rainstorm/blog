# Iterable 接口

实现 [Java™  8 - Iterable][1] 接口的类可以使用 for-each 循环， 1.8 的版本新增了两个接口，Java 集合框架里的所有集合都实现了这个接口。

本接口的 Demo 可在 [furry-octo-doodle](https://github.com/c-rainstorm/furry-octo-doodle) 仓库的 `me.rainstorm.fod.iterableTest` 中找到。

---

- [Iterable 接口](#iterable-%E6%8E%A5%E5%8F%A3)
  - [`iterator()`](#iterator)
    - [`ConcurrentModificationException`](#concurrentmodificationexception)
  - [`forEach(Consumer<? super T> action)`](#foreachconsumer-super-t-action)
  - [`spliterator()`](#spliterator)

---

## `iterator()`

返回迭代器。

实现了这个接口的类可以使用 [for-each][2] 遍历集合。

```java
@Test
public void test1() {
    for (Integer item : list) {
        LOGGER.debug(item);
    }
}

@Test
public void test2() {
    Integer[] rawArray = list.toArray(new Integer[0]);
    int sum = 0;
    for (int item : rawArray) {
        sum += item;
    }
    LOGGER.debug(sum);
}
```

### `ConcurrentModificationException`

在使用迭代器遍历集合的时候，不能进行任何变更集合（添加/删除等）的操作，因为这些操作会变更集合当前的版本号，如果版本号在遍历过程中被修改，在迭代遍历调用 `next` 方法时会检测到。

并发修改的 demo 【错误的使用方式】

```java
for (Integer item : list) {
    list.add(item);
}
```

`ArrayList` 遍历的 next 实现

```java
public E next() {
    // 检测遍历过程中的修改
    checkForComodification();
    int i = cursor;
    if (i >= size)
        throw new NoSuchElementException();
    Object[] elementData = ArrayList.this.elementData;
    if (i >= elementData.length)
        throw new ConcurrentModificationException();
    cursor = i + 1;
    return (E) elementData[lastRet = i];
}

final void checkForComodification() {
    // 如果集合被修改，抛异常
    if (modCount != expectedModCount)
        throw new ConcurrentModificationException();
}
```

如果我们想在迭代器遍历的时候删除指定元素，可以使用迭代器的 remove 方法。
下面是 `java.util.AbstractCollection#clear` 方法的默认实现。

```java
public void clear() {
    Iterator<E> it = iterator();
    while (it.hasNext()) {
        it.next();
        it.remove();
    }
}
```

下面是 ArrayList 迭代器的 remove 方法。

```java
public void remove() {
    if (lastRet < 0)
        throw new IllegalStateException();
    checkForComodification();

    try {
        // 调用正常的 remove 方法删除元素
        ArrayList.this.remove(lastRet);
        cursor = lastRet;
        lastRet = -1;
        // 因为迭代器在遍历的时候通过 expectedModCount 和 modCount 来判断内容是否有变动
        // 这个地方更新 expectedModCount 正常情况可以避免 ConcurrentModificationException
        expectedModCount = modCount;
    } catch (IndexOutOfBoundsException ex) {
        throw new ConcurrentModificationException();
    }
}
```

## `forEach(Consumer<? super T> action)`

**@since 1.8 这个方法从 JDK 1.8 开始引入**

对与集合中的每个元素，执行 action 所定义的操作，与 lambda 表达式结合可以写出很简洁的代码。

```java
@Test
public void test3() {
    // 对与 list 中的每一个元素，先打印数据元素的两倍，再打印一个 andThen
    list.forEach(((Consumer<Integer>) integer -> LOGGER.debug(integer * 2))
        .andThen(t -> LOGGER.debug("andThen")));
}
```

## `spliterator()`

**@since 1.8 这个方法从 JDK 1.8 开始引入**

总是推荐 Override 这个方法来提供一个更好的实现.

`Spliterator` 是 1.8 新增接口。主要用于将 list切分进行并行遍历。后续单独拆分出来进行学习。

[代码来源][3]

```java

ArrayList<String> list = new ArrayList<>();

list.add("A");
list.add("B");
list.add("C");
list.add("D");
list.add("E");
list.add("F");

Spliterator<String> spliterator1 = list.spliterator();
Spliterator<String> spliterator2 = spliterator1.trySplit();

spliterator1.forEachRemaining(System.out::println);

System.out.println("========");

spliterator2.forEachRemaining(System.out::println);

// output:
// D
// E
// F
// ========
// A
// B
// C
```

[1]:https://docs.oracle.com/javase/8/docs/api/java/lang/Iterable.html "Java™ Platform Standard Ed. 8 - Iterable"
[2]:https://docs.oracle.com/javase/8/docs/technotes/guides/language/foreach.html "The For-Each Loop"
[3]:https://howtodoinjava.com/java/collections/java-spliterator/ "Java Spliterator interface"
