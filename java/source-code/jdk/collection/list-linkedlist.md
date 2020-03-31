# LinkedList

## 时间复杂度

## 下标操作

任何带下标的操作操作基本都是 O(n/2) ~ O(n)

```java
/**
    * Returns the (non-null) Node at the specified element index.
    */
Node<E> node(int index) {
    // assert isElementIndex(index);

    if (index < (size >> 1)) {
        // 如果 index 小于 size 的一半，则从往后遍历
        Node<E> x = first;
        for (int i = 0; i < index; i++)
            x = x.next;
        return x;
    } else {
        // 如果 index 大于 size 的一半，则从后往前遍历
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x;
    }
}
```

## 双向链表

```java
/**
* Pointer to first node.
* Invariant: (first == null && last == null) ||
*            (first.prev == null && first.item != null)
*/
transient Node<E> first;

/**
* Pointer to last node.
* Invariant: (first == null && last == null) ||
*            (last.next == null && last.item != null)
*/
transient Node<E> last;

private static class Node<E> {
    E item;
    Node<E> next;
    Node<E> prev;

    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }
}
```

## 最后

没啥好说的很简单
