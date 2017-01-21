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

## 参考

1. [极客学院- java 反射机制](http://wiki.jikexueyuan.com/project/java-reflection/)
1. [Trail: The Reflection API](http://docs.oracle.com/javase/tutorial/reflect/index.html)