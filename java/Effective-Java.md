# Effective Java 学习笔记

<!-- TOC -->

- [Effective Java 学习笔记](#effective-java-学习笔记)
    - [第三章 对所有方法都通用的方法](#第三章-对所有方法都通用的方法)
        - [`equals()`](#equals)
        - [`hashCode()`](#hashcode)
    - [第六章 枚举和注解](#第六章-枚举和注解)
    - [第七章 方法](#第七章-方法)
    - [第八章 通用程序设计](#第八章-通用程序设计)

<!-- /TOC -->
---

## 第三章 对所有方法都通用的方法

### `equals()`

当类为值类（以类中保存的值来区别两个实例）时（枚举例外），需要重写 `equals()` 和 `hashCode()` 方法。

重写 `equals()` 需要遵守的约定：

1. 非空。`x != null`
1. 自反。`x.equals(x) == true`
1. 对称。`if(x.equals(y)) y.equals(x)`
1. 传递。`if(x.equals(y) && y.equals(z)) x.equals(z)`
1. 一致。 只要对象不变，每次调用必须能返回相同的结果。

Tips：
- 增加值组件时使用符合而不是继承。
```java
public class Point {
    private final int x;
    private final int y;

    public Point(int x, int y) {
        this.x = x;
        this.y = y;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) {
            return true;
        }
        if (o == null || !(o instanceof Point)) {
            return false;
        }

        Point point = (Point) o;
        return x == point.x && y == point.y;
    }

    @Override
    public int hashCode() {
        int result = x;
        result = 31 * result + y;
        return result;
    }
}

public class ColorPoint {
    private final Point point;
    private final Color color;

    public ColorPoint(Point point, Color color) {
        this.point = point;
        this.color = color;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) {
            return true;
        }
        if (o == null || !(o instanceof ColorPoint)) {
            return false;
        }

        ColorPoint that = (ColorPoint) o;
        if (point != null ? !point.equals(that.point) : that.point != null) {
            return false;
        }
        return color != null ? color.equals(that.color) : that.color == null;
    }

    @Override
    public int hashCode() {
        int result = point != null ? point.hashCode() : 0;
        result = 31 * result + (color != null ? color.hashCode() : 0);
        return result;
    }
}
```
- 对于 `float` 和 `double` 类型的值进行特殊处理 `Float.compare(f1, f2)`
- 域的比较顺序有可能会影响性能，所以应先比较最有可能不同的域及开销最低的域。
- 总是要覆盖 `hashCode()` 方法。

### `hashCode()`

约定：

1. 在应用程序执行过程中，只要 `equals()` 方法未更改，同一个对象调用 `hashCode()` 返回结果应该一致。
1. `if(x.equals(y)) x.hashCode() == y.hashCode();`

实现约定:

1. 将一个非 0 常量赋给 `result`。
1. 计算关键域的 hashCode
    - `boolean` --> `field ? 1 : 0;`
    - `Byte | char | short | int` --> `(int)field`
    - `long` --> `(int)(field ^ (field >>> 32))`
    - `float` --> `Float.floatToIntBits(field)`
    - `double` --> `Double.doubleToLongBits(field)` --> (long --> int)
    - 对象引用 --> 直接调用 `hashCode()`
    - 数组 --> `Arrays.hashCode()`;
1. `result = 31 * result + hashCode;`
1. 检验相等实例是否有相同 hashCode。

```java
public class PhoneNumber {
    private final short areaCode;
    private final short prefix;
    private final short lineNumber;

    public PhoneNumber(short areaCode, short prefix, short lineNumber) {
        rangeCheck(areaCode, 999, "area code");
        rangeCheck(prefix, 999, "prefix");
        rangeCheck(lineNumber, 9999, "lineNumber");
        this.areaCode = areaCode;
        this.prefix = prefix;
        this.lineNumber = lineNumber;
    }

    private void rangeCheck(int arg, int max, String name) {
        if(arg < 0 || arg > max){
            new IllegalArgumentException(name + ": " + arg);
        }
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) {
            return true;
        }
        if (o == null || !(o instanceof PhoneNumber)) {
            return false;
        }

        PhoneNumber that = (PhoneNumber) o;
        return lineNumber == that.lineNumber
                && areaCode == that.areaCode
                && prefix == that.prefix;
    }

    @Override
    public int hashCode() {
        int result = (int) areaCode;
        result = 31 * result + (int) prefix;
        result = 31 * result + (int) lineNumber;
        return result;
    }

    @Override
    public String toString() {
        return "PhoneNumber{" +
                "areaCode=" + areaCode +
                ", prefix=" + prefix +
                ", lineNumber=" + lineNumber +
                '}';
    }
}
```

Tips:

1. 删除冗余域。
1. 计算 hashCode 开销较大时，可以将其缓存到类内部。创建时计算或首次调用 `hashCode()` 时计算。

## 第六章 枚举和注解

1. 使用 `enum` 代替 int 常量
1. 用实例域代替序数。
    - 将特定枚举常量关联到特定的 int 值。
1. 使用 `EnumSet` 代替位域
    - 位域：使用或运算将多个常量合并到一个集合中。`text.applyStyles(STYLE_BOLD | STYLE_ITALIC)`
    - `text.applyStyles(EnumSet.of(Style.BOLD, Style.ITALIC))`
1. 使用 `EnumMap` 代替序数索引
    - `Map<Herb.Type, Set<Herb>> herbsByType = new EnumMap<Herb.Type, Set<Herb>>(Herb.Type.class);`
    - 多维：`EnumMap<..., EnumMap<...>>`

## 第七章 方法

1. 检查参数有效性
    - `public` 方法使用异常
    - `private` 方法使用断言
1. 必要时进行保护性拷贝
    - getXXX()
    - setXXX()
    - constructor
1. 谨慎设计方法签名
    - 命名方式要统一
    - 参数不宜超过 3 个。可以通过创建对象来传递多个参数
    - 参数类型优先使用接口
1. 慎用重载
    - 重载方法是在编译期确定的（静态绑定）
1. 慎用可变参数
1. 返回 0 长度的数组或集合，而不是 `null`
1. 为所有导出的 API 元素编写文档注释
    - [how to write doc comments](http://www.oracle.com/technetwork/articles/java/index-137868.html)


## 第八章 通用程序设计

1. 将局部变量的作用域最小化
    - 第一次使用时声明。
    - `for-each > for > while`
1. 对含有一组元素的数据结构，实现 `Iterable` 接口
1. 使用最合适的类型来存储数据
1. 使用 `StringBuilder` 来连接字符串
1. 通过接口引用对象（面向接口编程）
