# ArrayList

## 最大数组容量

> 有两种限制。
>
> 一是规范隐含的限制。
>
> Java数组的length必须是非负的int，
>
> 所以它的理论最大值就是java.lang.Integer.MAX_VALUE = 2^31-1 = 2147483647。
>
> 二是具体的实现带来的限制。
>
> 这会使得实际的JVM不一定能支持上面说的理论上的最大length。例如说如果有JVM使用uint32_t来记录对象大小的话，那可以允许的最大的数组长度（按元素的个数计算）就会是：
>
> (uint32_t的最大值 - 数组对象的对象头大小) / 数组元素大小
>
> 于是对于元素类型不同的数组，实际能创建的数组的最大length也会不同。JVM实现里可以有许多类似的例子会影响实际能创建的数组大小。

作者：RednaxelaFX
链接：https://www.zhihu.com/question/31809233/answer/53370790
来源：知乎
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

> The **count** must be of type int.
>
> if count is less than zero, the anewarray instruction throws a NegativeArraySizeException.
>
> [anewarray - Java Virtual Machine Specification][anewarray]

## 数组容量自增

```java
/**
    * The maximum size of array to allocate.
    * Some VMs reserve some header words in an array.
    * Attempts to allocate larger arrays may result in
    * OutOfMemoryError: Requested array size exceeds VM limit
    */
private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

private void grow(int minCapacity) {
    // overflow-conscious code
    int oldCapacity = elementData.length;
    // 正常一次扩容为原容量的 3/2
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    if (newCapacity - minCapacity < 0)
        // 新容量溢出，或者新容量依旧没有达到 minCapacity
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    // minCapacity is usually close to size, so this is a win:
    elementData = Arrays.copyOf(elementData, newCapacity);
}

private static int hugeCapacity(int minCapacity) {
    if (minCapacity < 0) // overflow
        throw new OutOfMemoryError();
    return (minCapacity > MAX_ARRAY_SIZE) ?
        Integer.MAX_VALUE :
        MAX_ARRAY_SIZE;
}
```

## 开源组件拓展

### `UidGenerator` 环形缓存

![CachedUidGenerator - 双RingBuffer](http://image.rainstorm.vip/blog/ringbuffer.png)

`@see` [UidGenerator][UidGenerator]

### `Disruptor` 高性能队列

// todo

## 参考

- [知乎: Java 数组有最大长度吗？ - 作者：RednaxelaFX](https://www.zhihu.com/question/31809233/answer/53370790)
- [anewarray - Java Virtual Machine Specification][anewarray]
- [关于面试题“ArrayList循环remove()要用Iterator”的研究 - since1986](https://juejin.im/post/59ef1ab0518825619a01e00a)
- [《我们一起进大厂》系列-ArrayList - 敖丙](https://juejin.im/post/5e14b51d5188253a9a213e83)
- [Java集合框架常见面试题 - JavaGuide](https://github.com/Snailclimb/JavaGuide/blob/master/docs/java/collection/Java%E9%9B%86%E5%90%88%E6%A1%86%E6%9E%B6%E5%B8%B8%E8%A7%81%E9%9D%A2%E8%AF%95%E9%A2%98.md#%E8%A1%A5%E5%85%85%E5%86%85%E5%AE%B9randomaccess%E6%8E%A5%E5%8F%A3)
- [Core algorithms deployed](https://cstheory.stackexchange.com/questions/19759/core-algorithms-deployed/19773#19773)
- [UidGenerator][UidGenerator]

[anewarray]: https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.anewarray
[UidGenerator]: https://github.com/baidu/uid-generator/blob/master/README.zh_cn.md
