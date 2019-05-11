# Integer 实现

`Integer` 是 int 的封装类型，本文分析其常用方法的特殊实现，（一般实现不再赘述，可自行阅读源码了解）

---

- [Integer 实现](#integer-%E5%AE%9E%E7%8E%B0)
  - [`Integer 缓存`](#integer-%E7%BC%93%E5%AD%98)
    - [源码分析](#%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90)
    - [测试样例](#%E6%B5%8B%E8%AF%95%E6%A0%B7%E4%BE%8B)
  - [`toString()`](#tostring)
    - [测试样例](#%E6%B5%8B%E8%AF%95%E6%A0%B7%E4%BE%8B-1)
  - [`highestOneBit()`](#highestonebit)
  - [`lowestOneBit()`](#lowestonebit)
  - [`rotateLeft()`](#rotateleft)
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

## `toString()`

借鉴 JDK 的写法我自己写了一下自己的 toString() 方法，测试结果见下个小节

```java
static void getChars(int i, int index, char[] buf) {
    int q, r;
    int charPos = index;
    char sign = 0;

    if (i < 0) {
      sign = '-';
      i = -i;
    }

    // Generate two digits per iteration
    while (i >= 65536) {
      // 每次处理两位数字
      q = i / 100;
    // really: r = i - (q * 100);
      r = i - ((q << 6) + (q << 5) + (q << 2));
      i = q;
      buf [--charPos] = DigitOnes[r];
      buf [--charPos] = DigitTens[r];
    }

    // Fall thru to fast mode for smaller numbers
    // assert(i <= 65536, i);
    for (;;) {
      // (double)52429/2^19 == 0.100000381469 相当于原来数值除以10
      q = (i * 52429) >>> (16+3);
      r = i - ((q << 3) + (q << 1));  // r = i-(q*10) ...
      buf [--charPos] = digits [r];
      i = q;
      if (i == 0) break;
    }
    if (sign != 0) {
      buf [--charPos] = sign;
    }
}
```

### 测试样例

完整测试代码可在 [furry-octo-doodle](https://github.com/c-rainstorm/furry-octo-doodle) 仓库的 `me.rainstorm.fod.lang.IntegerTest` 中找到。

测试机器：MacBook Pro (15-inch, 2017) 2.8 GHz Intel Core i7, 16 GB 2133 MHz LPDDR3

```java
private static void getChars(int i, int index, char[] buf) {
    int q, r;
    int charPos = index;

    if (i < 0) {
        i = -i;
        buf[0] = '-';
    }

    while (i != 0) {
        // 每一位统一除 10
        q = i / 10;
        r = i - ((q << 3) + (q << 1));
        i = q;
        buf[--charPos] = DIGIT_ONES[r];
    }
}
```

每个量级测10次，去掉一个最大值，去掉一个最小值，剩余8次取平均值，单位为毫秒

代码平均耗时 - int 上界 65535
|代码|100|1000|10000|100000|1000000|10000000|100000000|
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
|JDK|1-2|1-2|6.5|25.125|86.875|437.5|3042.125|
|自己的实现|0-1|1-2|6.37|21.25|85.25|449.125|3603.875|

代码平均耗时 - int 上界 Integer.MAX_VALUE
|代码|100|1000|10000|100000|1000000|10000000|100000000|
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
|JDK|0-1|1.625|8.5|40.375|88.875|644.875|4210.75|
|自己的实现|0-1|1.875|8.125|26.5|113|631.5|4777.875|

## `highestOneBit()`

```java
public static int highestOneBit(int i) {
    // HD, Figure 3-1
    // 最高位右侧全部置为1
    i |= (i >>  1);
    i |= (i >>  2);
    i |= (i >>  4);
    i |= (i >>  8);
    i |= (i >> 16);
    // 去掉最高位右侧所有1
    return i - (i >>> 1);
}
```

## `lowestOneBit()`

获取最低位。利用的是补码的性质。

最低位的1 右侧必然全部为0，取负数以后最低位的1置为0，右侧0全部置为1，然后加一，所有1都会进位，然后原来最低位的1 又从0变为1了，取 & 操作，可以去掉前面的所有 1，从而得到最低位的1.
举个例子

2
2 的二进制是  10
-2 的二进制是  1111110 （高位补1）

2 & -2 = 2

```java
public static int lowestOneBit(int i) {
    // HD, Section 2-1
    return i & -i;
}
```

## `rotateLeft()`

```java
public static int rotateLeft(int i, int distance) {
    // i << distance 得到新值的高 32-dis 位
    // i >>> -distance == i >>> (32 - distance)，得到新增低 distance 位
    return (i << distance) | (i >>> -distance);
}
```

## 参考

1. [[FYI] 关于Integer的自动缓存大小 - RednaxelaFX](https://rednaxelafx.iteye.com/blog/680746)
2. [java.lang.Integer源码精读(一)](https://www.jianshu.com/p/02c1d9092347)
