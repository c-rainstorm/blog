# java.lang 学习笔记

包描述：
- 提供 java 语言最基础的类。最重要的类有两个：一个是 `Object`, 它是 java 类继承树的根源；一个是 `Class`，运行时加载到内存的所有类都是其子类。
- 封装基本类型 && `Void`
- `Math` && `String`、`StringBuffer`、`StringBuilder`.
- `ClassLoader`, `Process`, `ProcessBuilder`, `Runtime`, `SecurityManager`, `System`。动态加载类，创建另外的进程， 主机环境。执行安全策略。
- `Throwable`， 子类为可抛异常或错误。
- `java.nio.charset.Charset` 类中描述了 java 平台字符集的命名规范，以及一系列必须支持的标准编码。

---

## `Object`

1. `equals() && hashCode()`
    - 详情参见 『[Effective-Java.md](../reading-notes/Effective-Java.md)』
1. `clone() && finalize()`
    - 不覆盖，不使用
1. `toString()`
    - 默认值 `getClass().getName() + '@' + Integer.toHexString(hashCode())`
    - 一定覆盖

## `Class`

- 加载类
    - `forName()`
- 可以通过该类获取本类的基本信息
    - `getAnnotations()  //获取注解`
    - `getConstructors()  //获取所有构造器`
    - `getDeclaredFields()  //获取定义的域`
    - `getPackage() //获取包`
    - `isEnum()  // 是否是枚举类`
    - `getName() // 获取全类名`

## `Void`

- `Void.TYPE == void`

## 封装类型

### 共性

- 获取类类型 `Double.TYPE`
- 包含方法
    1. 基本类型转封装类型
        - 构造器
        - `valueOf()`
    1. 字符串转封装类型
        - 构造器
        - `valueOf()`
        - `parseXXX()`
    1. 封装类型转基本类型
        - `XXXValue()`
    1. 封装类型转字符串
        - `toString()`
    1. 比较
        - `compare(T x, T y)  // T 为基本类型`
        - `compareTo(T another) // T 为封装类型`
    1. 类型转换
        - API 为我们提供了方便的转换函数。可以将 `byte`、`short`、`int`、`long`、`float`、`double` 类型的数据进行相互转换，具体的函数名请查阅 API 文档。
    1. 进制转换
        - `decode()` 将八进制、十进制、十六进制字符串转为相应类型（字符串格式有要求，请参考 API 文档）
        - `toBinaryString()  // 10 --> 2`  // 以下方法只在 `Integer` 和 `Long` 中有。 
        - `toHexString()     // 16 --> 16`    
        - `toOctalString()   // 8 --> 10`

### 特性

- `Boolean`
    - 提供了两个 `Boolean` 类型的逻辑操作（and、or、xor）
- `Long && Integer`
    - 位操作
        - `highestOneBit()`。 二进制只保留最高位的 1.
            - 负数都为 `Integer.MIN_VALUE`; 
            - 0 为 `0`; 
            - 正数都为 `2^n`. 例子： `Integer.highestOneBit(10) == 8`, n 为最高一个 1 的位数。
        - `lowestOneBit()` 与上一个相对，二进制只保留最低位的 1。
        - `numberOfLeadingZeros()` 二进制前导 0 的个数
        - `numberOfTrailingZeros()` 二进制尾部 0 的个数
        - `reverse()` 二进制反转
        - `reverseBytes()` 二进制反转，以 `byte` 为单位。
        - `rotateLeft()` 左旋
        - `rotateRight()` 右旋
- `Double && Float`
    - `floatToIntBits()`  // float 数据位模式的 int 表示。  **以下两个方法在求 float 数据和 double 数据的 hashCode 时会经常使用**
    - `doubleToLongBits()` // double 数据位模式的 long 表示
    - `intBitsToFloat()`
    - `longBitsToDouble()`
    - `isFinite()` // 是有限小数
    - `isInFinite()` // 是无穷大
    - `isNaN()`    // 不是数
- `Character`
    
    > The Java platform uses the UTF-16 representation in char arrays and in the String and StringBuffer classes. In this representation, supplementary characters are represented as a pair of char values, the first from the high-surrogates range, (\uD800-\uDBFF), the second from the low-surrogates range (\uDC00-\uDFFF).

    - API 中包含了很多类似与正则表达式判断的方法。
    - 对代码点的处理等等

- `Math && StrictMath`
    - 很多时候 `Math` 只是简单调用 `StrictMath` 中的方法
    - `Math` 提供了更好地性能。
    - `StrictMath` 提供了一致性的结果。



## 参考

1. [极客学院- java 反射机制](http://wiki.jikexueyuan.com/project/java-reflection/)
1. [Trail: The Reflection API](http://docs.oracle.com/javase/tutorial/reflect/index.html)