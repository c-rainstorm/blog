# Caffeine Wiki 中文版

- [Caffeine Wiki 中文版](#caffeine-wiki-%e4%b8%ad%e6%96%87%e7%89%88)
  - [简介](#%e7%ae%80%e4%bb%8b)
  - [缓存](#%e7%bc%93%e5%ad%98)
    - [添加](#%e6%b7%bb%e5%8a%a0)
    - [驱逐](#%e9%a9%b1%e9%80%90)
      - [基于容量的驱逐](#%e5%9f%ba%e4%ba%8e%e5%ae%b9%e9%87%8f%e7%9a%84%e9%a9%b1%e9%80%90)
      - [基于时间的驱逐](#%e5%9f%ba%e4%ba%8e%e6%97%b6%e9%97%b4%e7%9a%84%e9%a9%b1%e9%80%90)
      - [基于引用的驱逐](#%e5%9f%ba%e4%ba%8e%e5%bc%95%e7%94%a8%e7%9a%84%e9%a9%b1%e9%80%90)
    - [移除](#%e7%a7%bb%e9%99%a4)
    - [刷新](#%e5%88%b7%e6%96%b0)
    - [写出](#%e5%86%99%e5%87%ba)
    - [统计](#%e7%bb%9f%e8%ae%a1)
  - [拓展](#%e6%8b%93%e5%b1%95)
    - [模拟器](#%e6%a8%a1%e6%8b%9f%e5%99%a8)
    - [JCache](#jcache)
    - [Guava](#guava)
  - [性能](#%e6%80%a7%e8%83%bd)
    - [设计](#%e8%ae%be%e8%ae%a1)
    - [效率](#%e6%95%88%e7%8e%87)
    - [基准测试](#%e5%9f%ba%e5%87%86%e6%b5%8b%e8%af%95)
    - [内存开销](#%e5%86%85%e5%ad%98%e5%bc%80%e9%94%80)
  - [参考](#%e5%8f%82%e8%80%83)

## 简介

Caffeine 是基于 Java 8 的[高性能]((#%e5%9f%ba%e5%87%86%e6%b5%8b%e8%af%95))缓存库，提供了[近乎最优](#%e6%95%88%e7%8e%87)的命中率。

缓存和 [`ConcurrentMap`][ConcurrentMap] 很像，但是还是有区别的。最大的不同在于 [`ConcurrentMap`][ConcurrentMap] 一直持有已添加的所有元素，直到元素被显式移除。缓存通常有一定的驱逐策略，在内存不足时来自动驱逐条目。在一些场景下，`LoadingCache` 或者 `AsyncLoadingCache` 的自动缓存加载非常有用

Caffeine 提供了灵活的 API 来创建组合以下特性的缓存：

- 支持[自动加载条目](#%e6%b7%bb%e5%8a%a0)到缓存，支持异步加载。
- 达到最大容量时基于 [频率和新鲜度](#%e6%95%88%e7%8e%87) 进行 [基于容量的驱逐](#%e5%9f%ba%e4%ba%8e%e5%ae%b9%e9%87%8f%e7%9a%84%e9%a9%b1%e9%80%90)
- 条目超过一定时间未访问和写入，进行 [基于时间的驱逐](#%e5%9f%ba%e4%ba%8e%e6%97%b6%e9%97%b4%e7%9a%84%e9%a9%b1%e9%80%90)
- 条目过期[异步刷新](#%e5%88%b7%e6%96%b0)
- key 自动使用 [弱引用](#%e5%9f%ba%e4%ba%8e%e5%bc%95%e7%94%a8%e7%9a%84%e9%a9%b1%e9%80%90) 封装
- value 自动使用 [弱引用或软引用](#%e5%9f%ba%e4%ba%8e%e5%bc%95%e7%94%a8%e7%9a%84%e9%a9%b1%e9%80%90) 封装
- 条目驱逐或移除[通知](#%e7%a7%bb%e9%99%a4)
- [写入更新](#%e5%86%99%e5%87%ba)到外部资源
- 累计缓存访问[统计](#%e7%bb%9f%e8%ae%a1)

为了提高整合度，在插件模块提供了 [`JSR-10 JCache`](#jcache) 和 [`Guava`](#guava) 的适配器。

JSR-107 基于 Java 6 的API，以牺牲功能和性能为代价，最大限度地减少了缓存供应商特定的代码。 Guava 的 Cache 是​​ Caffeine 的原型库，适配器提供了一种简单的迁移策略。

## 缓存

### 添加

### 驱逐

#### 基于容量的驱逐

#### 基于时间的驱逐

#### 基于引用的驱逐

### 移除

### 刷新

### 写出

### 统计

## 拓展

### 模拟器

### JCache

### Guava

## 性能

### 设计

### 效率

### 基准测试

### 内存开销

## 参考

- [ben-manes/caffeine][ben-manes/caffeine]

[ben-manes/caffeine]: https://github.com/ben-manes/caffeine/wiki
[ConcurrentMap]:https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ConcurrentMap.html
