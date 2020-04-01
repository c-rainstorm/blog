# `EnumMap<K extends Enum<K>, V>`

## 设计亮点

key-value 使用数组代替普通的哈希表，很赞的设计。

```java
/**
    * All of the values comprising K.  (Cached for performance.)
    */
private transient K[] keyUniverse;

/**
    * Array representation of this map.  The ith element is the value
    * to which universe[i] is currently mapped, or null if it isn't
    * mapped to anything, or NULL if it's mapped to null.
    */
private transient Object[] vals;
```
