# HashMap 详解

本文从最常使用的场景分析 HashMap 的源码实现

---

- [HashMap 详解](#hashmap-%E8%AF%A6%E8%A7%A3)
  - [简述](#%E7%AE%80%E8%BF%B0)
  - [容量变更](#%E5%AE%B9%E9%87%8F%E5%8F%98%E6%9B%B4)
  - [普通链和树的转换](#%E6%99%AE%E9%80%9A%E9%93%BE%E5%92%8C%E6%A0%91%E7%9A%84%E8%BD%AC%E6%8D%A2)
    - [链转成树](#%E9%93%BE%E8%BD%AC%E6%88%90%E6%A0%91)
    - [树转链](#%E6%A0%91%E8%BD%AC%E9%93%BE)
  - [`put`](#put)
  - [`get`](#get)
  - [`remove`](#remove)
  - [遍历](#%E9%81%8D%E5%8E%86)

---

## 简述

HashMap 的容量是 2 的 n 次方，在计算key 对应的下标时使用 （容量-1）来做 mask，扩容缩容的时候也会注意变动后容量满足这个条件。

## 容量变更

```java
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    if (oldCap > 0) {
        //HashMap(int initialCapacity) 第一次添加元素
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            // 容量和 resize 阈值翻倍
            newThr = oldThr << 1; // double threshold
    }
    else if (oldThr > 0) // initial capacity was placed in threshold
        // new HashMap(Map<? extends K, ? extends V> m) 第一次添加元素
        newCap = oldThr;
    else {               // zero initial threshold signifies using defaults
        // new HashMap() 第一次添加元素扩容
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            // 遍历每一个 table 每一个位置
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                if (e.next == null)
                    // 如果当前位置只有一个元素，直接添加到新表
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode)
                    // 如果是树形结构，特殊处理，稍后分析
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // preserve order
                    // 先构造在原下标和新下标(原下标+原容量)的两条链，然后设置到 table对应位置
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        if ((e.hash & oldCap) == 0) {
                            // e.hash & oldCap == 0 说明，这个元素在新数组的下标不变
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        else {
                            // e.hash & oldCap == 1 说明，这个元素在新数组的下标为(当前下标+原容量)
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    if (loTail != null) {
                        // 原下标位置有元素，配置一下
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) {
                        // 新下标的位置有元素，配置一下
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```

容量变更过程中树的切分

```java
final void split(HashMap<K,V> map, Node<K,V>[] tab, int index, int bit) {
    TreeNode<K,V> b = this;
    // Relink into lo and hi lists, preserving order
    TreeNode<K,V> loHead = null, loTail = null;
    TreeNode<K,V> hiHead = null, hiTail = null;
    int lc = 0, hc = 0;
    for (TreeNode<K,V> e = b, next; e != null; e = next) {
        next = (TreeNode<K,V>)e.next;
        e.next = null;
        if ((e.hash & bit) == 0) {
            // e.hash & bit == 0 说明，这个元素在新数组的下标不变
            if ((e.prev = loTail) == null)
                loHead = e;
            else
                loTail.next = e;
            loTail = e;
            ++lc;
        }
        else {
             // e.hash & bit == 0 说明，这个元素在新数组的下标为(当前下标+原容量)
            if ((e.prev = hiTail) == null)
                hiHead = e;
            else
                hiTail.next = e;
            hiTail = e;
            ++hc;
        }
    }

    if (loHead != null) {
        if (lc <= UNTREEIFY_THRESHOLD)
            // todo
            tab[index] = loHead.untreeify(map);
        else {
            tab[index] = loHead;
            if (hiHead != null) // (else is already treeified)
                // todo
                loHead.treeify(tab);
        }
    }
    if (hiHead != null) {
        if (hc <= UNTREEIFY_THRESHOLD)
            tab[index + bit] = hiHead.untreeify(map);
        else {
            tab[index + bit] = hiHead;
            if (loHead != null)
                hiHead.treeify(tab);
        }
    }
}
```

## 普通链和树的转换

### 链转成树

```java
/**
 * 链转成树
 * 1. 如果当前的 HashMap 的table 容量过小（小于 MIN_TREEIFY_CAPACITY（64）），则对哈希表进行扩容操作
 * 2. 否则进行转树操作
 *
 */
final void treeifyBin(Node<K,V>[] tab, int hash) {
    int n, index; Node<K,V> e;
    if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
        // table 容量过小，翻倍扩容
        resize();
    else if ((e = tab[index = (n - 1) & hash]) != null) {
        // 将单链转成树
        TreeNode<K,V> hd = null, tl = null;
        do {
            // 跟据原节点创建对应的树节点
            // 将这些树节点根据 prev 和 next 构造成链
            TreeNode<K,V> p = replacementTreeNode(e, null);
            if (tl == null)
                hd = p;
            else {
                p.prev = tl;
                tl.next = p;
            }
            tl = p;
        } while ((e = e.next) != null);
        if ((tab[index] = hd) != null)
            // 链转树
            hd.treeify(tab);
    }
}

final void treeify(Node<K,V>[] tab) {
    TreeNode<K,V> root = null;
    for (TreeNode<K,V> x = this, next; x != null; x = next) {
        // 遍历已构造好的 TreeNode 链
        next = (TreeNode<K,V>)x.next;
        x.left = x.right = null;
        if (root == null) {
            // 第一个节点暂时作为 root 节点
            x.parent = null;
            x.red = false;
            root = x;
        }
        else {
            K k = x.key;
            int h = x.hash;
            Class<?> kc = null;
            for (TreeNode<K,V> p = root;;) {
                int dir, ph;
                K pk = p.key;
                if ((ph = p.hash) > h)
                    dir = -1;
                else if (ph < h)
                    dir = 1;
                else if ((kc == null &&
                          (kc = comparableClassFor(k)) == null) ||
                         (dir = compareComparables(kc, k, pk)) == 0)
                    dir = tieBreakOrder(k, pk);

                TreeNode<K,V> xp = p;
                if ((p = (dir <= 0) ? p.left : p.right) == null) {
                    x.parent = xp;
                    if (dir <= 0)
                        xp.left = x;
                    else
                        xp.right = x;
                    root = balanceInsertion(root, x);
                    break;
                }
            }
        }
    }
    moveRootToFront(tab, root);
}
```

### 树转链

## `put`

```java
// put key-value 对到 hashmap
public V put(K key, V value) {
    // 先计算 key 的 hash
    return putVal(hash(key), key, value, false, true);
}

// 如果 key 为 null,返回 0，说明 hashMap 是允许 key 为 null 的
// 不为 null 的话，调用 key 的 hashCode 方法，hash 结果与其高16位做异或，是要尽量避免因为
// 计算下标使用 （容量-1）做 mask，防止掩码处理部分相同，掩码以外部分不同时必然发生的 hash碰撞
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}

final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0)
        // table 为null，或长度为0说明 尚未分配，初始化 table
        n = (tab = resize()).length;

    // n 是容量，n-1 为下标计算掩码
    if ((p = tab[i = (n - 1) & hash]) == null)
        // 如果这个下标位置还没有元素，创建节点直接添加
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            // 链上第一个元素 key 的引用与当前key相同或 equals 判定 key 相等
            e = p;
        else if (p instanceof TreeNode)
            // 如果已经转成树了，使用树的 api 添加元素，稍后单独分析 todo
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    // 到达链尾部，未找到已存在节点，创建新节点
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        // 如果当前链节点数量到达临界值，转成树
                        treeifyBin(tab, hash);
                    break;
                }
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    // 找到了已存在的节点
                    break;
                p = e;
            }
        }
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    // 变更版本
    ++modCount;
    if (++size > threshold)
        // threshold 是扩容阈值，计算方式是 容量*装载系数
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

## `get`

## `remove`

## 遍历
