# Integer 实现

`Integer` 是 int 的封装类型，本文分析其常用方法的特殊实现，（一般实现不再赘述，可自行阅读源码了解）

---

- [Integer 实现](#integer-%E5%AE%9E%E7%8E%B0)
  - [`Integer 缓存`](#integer-%E7%BC%93%E5%AD%98)
    - [源码分析](#%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90)
    - [测试样例](#%E6%B5%8B%E8%AF%95%E6%A0%B7%E4%BE%8B)
  - [参考](#%E5%8F%82%E8%80%83)

---

## `Integer 缓存`

Integer 会缓存一部分经常使用的整数来减少创建对象的开销，我们用 `[low, high]` 的形式代表 Integer 缓存的整数范围。

默认情况下，low = -128, high = 127, 一共 256 个整数。在查看源码的时候发现这个范围是可配置的。

low = -128 是定死的，这个不能变动，但是我们可以配置 high 来缓存更大范围的整数(如果配置值小于 127 默认还使用127)

配置 high 有两种方式

1. `-server -XX:AutoBoxCacheMax=？` JVM 参数给我们了一个入口，让我们可以指定 high 值,  AutoBoxCacheMax 是我们想要配置的 high 值，
2. `-server -XX:+AggressiveOpts`, 如果打开开关 AutoBoxCacheMax 不给值，默认 `high=20000`。
3. 命令行添加系统属性 `-Djava.lang.Integer.IntegerCache.high=?` 这种方式 client 和 server 模式都可以使用
4. 另：如果 `AutoBoxCacheMax 和 -Djava.lang.Integer.IntegerCache.high` 同时出现在命令行参数上，以 `AutoBoxCacheMax` 为准。

### 源码分析

```java

private static class IntegerCache {
  // 缓存下界
  static final int low = -128;
  // 缓存上界
  static final int high;
  // 缓存数组
  static final Integer cache[];

  static {
      // high value may be configured by property
      int h = 127;
      // 从 JVM 获取系统属性
      String integerCacheHighPropValue =
          sun.misc.VM.getSavedProperty("java.lang.Integer.IntegerCache.high");
      if (integerCacheHighPropValue != null) {
          try {
              int i = parseInt(integerCacheHighPropValue);
              i = Math.max(i, 127);
              // Maximum array size is Integer.MAX_VALUE
              h = Math.min(i, Integer.MAX_VALUE - (-low) -1);
          } catch( NumberFormatException nfe) {
              // If the property cannot be parsed into an int, ignore it.
              // 格式错误使用默认值
          }
      }
      high = h;

      cache = new Integer[(high - low) + 1];
      int j = low;
      for(int k = 0; k < cache.length; k++)
          cache[k] = new Integer(j++);

      // range [-128, 127] must be interned (JLS7 5.1.7)
      assert IntegerCache.high >= 127;
  }

  private IntegerCache() {}
}
```

下面是 JDK1.8 的 hotspot 的相关代码，用来配置上界

```cpp
// hotspot/src/share/vm/runtime/arguments.cpp

// Aggressive optimization flags  -XX:+AggressiveOpts
void Arguments::set_aggressive_opts_flags() {
// COMPILER2 代表是 -server 模式启动， COMPILER1 是 -client 模式启动
// 也就是说这个 -XX:AutoBoxCacheMax=？这个JVM 参数在 server 模式下才是有效的。
#ifdef COMPILER2
  // ...
  if (AggressiveOpts || !FLAG_IS_DEFAULT(AutoBoxCacheMax)) {
    if (FLAG_IS_DEFAULT(EliminateAutoBox)) {
      FLAG_SET_DEFAULT(EliminateAutoBox, true);
    }
    if (FLAG_IS_DEFAULT(AutoBoxCacheMax)) {
      FLAG_SET_DEFAULT(AutoBoxCacheMax, 20000);
    }

    // Feed the cache size setting into the JDK
    char buffer[1024];
    jio_snprintf(buffer, 1024, "java.lang.Integer.IntegerCache.high=" INTX_FORMAT, AutoBoxCacheMax);
    add_property(buffer);
  }

  // ...
#endif
}
```

### 测试样例

[[FYI] 关于Integer的自动缓存大小 - RednaxelaFX](https://rednaxelafx.iteye.com/blog/680746) 里有详细的测试 case，不再单独写一次

样例补充：

```bash
java -server -Djava.lang.Integer.IntegerCache.high=1001 -XX:AutoBoxCacheMax=1000 me.rainstorm.fod.lang.TestAutoBoxCache
true
false
false
```

```bash
java -server -XX:AutoBoxCacheMax=1000 -Djava.lang.Integer.IntegerCache.high=1001 me.rainstorm.fod.lang.TestAutoBoxCache
true
false
false
```

## 参考

[[FYI] 关于Integer的自动缓存大小 - RednaxelaFX](https://rednaxelafx.iteye.com/blog/680746)
