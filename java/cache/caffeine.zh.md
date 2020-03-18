# Caffeine Wiki 中文版

- [Caffeine Wiki 中文版](#caffeine-wiki-%e4%b8%ad%e6%96%87%e7%89%88)
  - [简介](#%e7%ae%80%e4%bb%8b)
  - [缓存](#%e7%bc%93%e5%ad%98)
    - [添加](#%e6%b7%bb%e5%8a%a0)
      - [手动加载](#%e6%89%8b%e5%8a%a8%e5%8a%a0%e8%bd%bd)
      - [自动加载](#%e8%87%aa%e5%8a%a8%e5%8a%a0%e8%bd%bd)
      - [手动异步加载](#%e6%89%8b%e5%8a%a8%e5%bc%82%e6%ad%a5%e5%8a%a0%e8%bd%bd)
      - [异步自动加载](#%e5%bc%82%e6%ad%a5%e8%87%aa%e5%8a%a8%e5%8a%a0%e8%bd%bd)
    - [驱逐](#%e9%a9%b1%e9%80%90)
      - [基于容量的驱逐](#%e5%9f%ba%e4%ba%8e%e5%ae%b9%e9%87%8f%e7%9a%84%e9%a9%b1%e9%80%90)
      - [基于时间的驱逐](#%e5%9f%ba%e4%ba%8e%e6%97%b6%e9%97%b4%e7%9a%84%e9%a9%b1%e9%80%90)
      - [基于引用的驱逐](#%e5%9f%ba%e4%ba%8e%e5%bc%95%e7%94%a8%e7%9a%84%e9%a9%b1%e9%80%90)
    - [移除](#%e7%a7%bb%e9%99%a4)
      - [显式删除](#%e6%98%be%e5%bc%8f%e5%88%a0%e9%99%a4)
      - [移除监听器](#%e7%a7%bb%e9%99%a4%e7%9b%91%e5%90%ac%e5%99%a8)
    - [刷新](#%e5%88%b7%e6%96%b0)
    - [Writer](#writer)
      - [可能的用例](#%e5%8f%af%e8%83%bd%e7%9a%84%e7%94%a8%e4%be%8b)
        - [写模式](#%e5%86%99%e6%a8%a1%e5%bc%8f)
        - [分层](#%e5%88%86%e5%b1%82)
        - [同步监听器](#%e5%90%8c%e6%ad%a5%e7%9b%91%e5%90%ac%e5%99%a8)
    - [统计](#%e7%bb%9f%e8%ae%a1)
    - [清理](#%e6%b8%85%e7%90%86)
    - [策略](#%e7%ad%96%e7%95%a5)
      - [基于大小](#%e5%9f%ba%e4%ba%8e%e5%a4%a7%e5%b0%8f)
      - [基于时间](#%e5%9f%ba%e4%ba%8e%e6%97%b6%e9%97%b4)
    - [测试](#%e6%b5%8b%e8%af%95)
    - [FAQ](#faq)
      - [固定条目](#%e5%9b%ba%e5%ae%9a%e6%9d%a1%e7%9b%ae)
      - [递归计算](#%e9%80%92%e5%bd%92%e8%ae%a1%e7%ae%97)
      - [写争夺](#%e5%86%99%e4%ba%89%e5%a4%ba)
  - [性能](#%e6%80%a7%e8%83%bd)
    - [设计](#%e8%ae%be%e8%ae%a1)
      - [访问操作有序队列](#%e8%ae%bf%e9%97%ae%e6%93%8d%e4%bd%9c%e6%9c%89%e5%ba%8f%e9%98%9f%e5%88%97)
      - [写操作有序队列](#%e5%86%99%e6%93%8d%e4%bd%9c%e6%9c%89%e5%ba%8f%e9%98%9f%e5%88%97)
      - [分层的计时器轮](#%e5%88%86%e5%b1%82%e7%9a%84%e8%ae%a1%e6%97%b6%e5%99%a8%e8%bd%ae)
      - [读缓冲区](#%e8%af%bb%e7%bc%93%e5%86%b2%e5%8c%ba)
      - [写缓冲区](#%e5%86%99%e7%bc%93%e5%86%b2%e5%8c%ba)
      - [锁开销均摊](#%e9%94%81%e5%bc%80%e9%94%80%e5%9d%87%e6%91%8a)
      - [条目状态转换](#%e6%9d%a1%e7%9b%ae%e7%8a%b6%e6%80%81%e8%bd%ac%e6%8d%a2)
      - [代码生成](#%e4%bb%a3%e7%a0%81%e7%94%9f%e6%88%90)
      - [被装饰的哈希表](#%e8%a2%ab%e8%a3%85%e9%a5%b0%e7%9a%84%e5%93%88%e5%b8%8c%e8%a1%a8)
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
- [写入更新](#writer)到外部资源
- 累计缓存访问[统计](#%e7%bb%9f%e8%ae%a1)

为了提高整合度，在插件模块提供了 [`JSR-10 JCache`](#jcache) 和 [`Guava`](#guava) 的适配器。

JSR-107 基于 Java 6 的API，以牺牲功能和性能为代价，最大限度地减少了缓存供应商特定的代码。 Guava 的 Cache 是​​ Caffeine 的原型库，适配器提供了一种简单的迁移策略。

## 缓存

### 添加

Caffeine 提供了四种添加缓存的策略：手动手动、自动加载，手动异步，自动异步。

#### 手动加载

```java
Cache<Key, Graph> cache = Caffeine.newBuilder()
    .expireAfterWrite(10, TimeUnit.MINUTES)
    .maximumSize(10_000)
    .build();

// Lookup an entry, or null if not found
Graph graph = cache.getIfPresent(key);
// Lookup and compute an entry if absent, or null if not computable
graph = cache.get(key, k -> createExpensiveGraph(key));
// Insert or update an entry
cache.put(key, graph);
// Remove an entry
cache.invalidate(key);
```

`Cache` 接口允许对缓存条目显式的获取、更新、失效。

条目可以通过 `cache.put(key, value)` 直接插入缓存。这会覆盖指定 key 的之前值。可以使用 `cache.get(key, k -> value)` 来自动计算并插入缓存防止和其他写操作竞争。 注意 `cache.get` 在 条目不可计算或者计算失败时会返回 `null`。

`Cache.asMap()` 暴露的视图也可以用来修改缓存。

#### 自动加载

```java
LoadingCache<Key, Graph> cache = Caffeine.newBuilder()
    .maximumSize(10_000)
    .expireAfterWrite(10, TimeUnit.MINUTES)
    .build(key -> createExpensiveGraph(key));

// Lookup and compute an entry if absent, or null if not computable
Graph graph = cache.get(key);
// Lookup and compute entries that are absent
Map<Key, Graph> graphs = cache.getAll(keys);
```

`LoadingCache = Cache + CacheLoader`。

批量查询可以用 `getAll`。默认情况下，`getAll` 会对每个缺失的 key 单独调用 `CacheLoader.load`，当 批量查询比很多单独查询要高效时，可以覆盖 `CacheLoader.loadAll` 方法。

请注意，您可以编写一个 `CacheLoader.loadAll` 实现，该实现加载未明确请求的键的值。比如，如果计算某个组中任何键的值时获取到了该组中所有键的值，则 `loadAll` 可能会同时加载该组中的其余键。

#### 手动异步加载

```java
AsyncCache<Key, Graph> cache = Caffeine.newBuilder()
    .expireAfterWrite(10, TimeUnit.MINUTES)
    .maximumSize(10_000)
    .buildAsync();

// Lookup and asynchronously compute an entry if absent
CompletableFuture<Graph> graph = cache.get(key, k -> createExpensiveGraph(key));
```

`AsyncCache` 使用 `[Executor][Executor]` 计算条目，并返回一个 [CompletableFuture]。这可以使用响应式编程的方式来使用缓存。

`synchronous()` 视图会阻塞缓存知道异步计算结束。

`AsyncCache.asMap()` 暴露的视图也可以用来修改缓存。

默认执行器是 [`ForkJoinPool.commonPool()`][ForkJoinPool]，可以通过 `Caffeine.executor(Executor)` 进行自定义。

#### 异步自动加载

```java
AsyncLoadingCache<Key, Graph> cache = Caffeine.newBuilder()
    .maximumSize(10_000)
    .expireAfterWrite(10, TimeUnit.MINUTES)
    // Either: Build with a synchronous computation that is wrapped as asynchronous
    .buildAsync(key -> createExpensiveGraph(key));
    // Or: Build with a asynchronous computation that returns a future
    .buildAsync((key, executor) -> createExpensiveGraphAsync(key, executor));

// Lookup and asynchronously compute an entry if absent
CompletableFuture<Graph> graph = cache.get(key);
// Lookup and asynchronously compute entries that are absent
CompletableFuture<Map<Key, Graph>> graphs = cache.getAll(keys);
```

`AsyncLoadingCache = AsyncCache + AsyncCacheLoader`。

`CacheLoader` 用于同步加载，`AsyncCacheLoader` 用于异步加载，此时可以拿到 `[CompletableFuture][CompletableFuture]`。

### 驱逐

Caffeine 提供了三种驱逐策略：基于容量、基于时间、基于引用的。

#### 基于容量的驱逐

```java
// Evict based on the number of entries in the cache
LoadingCache<Key, Graph> graphs = Caffeine.newBuilder()
    .maximumSize(10_000)
    .build(key -> createExpensiveGraph(key));

// Evict based on the number of vertices in the cache
LoadingCache<Key, Graph> graphs = Caffeine.newBuilder()
    .maximumWeight(10_000)
    .weigher((Key key, Graph graph) -> graph.vertices().size())
    .build(key -> createExpensiveGraph(key));
```

如果你的缓存不应该超过指定大小，使用`Caffeine.maximumSize(long)`。缓存会使用 [LRU](#%e6%95%88%e7%8e%87) 进行自动驱逐。

如果不同的条目有不同的权重，比如不同的值占用内存不同，你可以指定一个权重函数，`Caffeine.weigher(Weigher)` 和 `Caffeine.maximumWeight(long)` 配合来达到基于容量和权重的驱逐。

计算权重发生在创建和更新操作时，之后就是静态的数据了，并且在进行驱逐选择时不使用相对权重。

#### 基于时间的驱逐

```java
// Evict based on a fixed expiration policy
LoadingCache<Key, Graph> graphs = Caffeine.newBuilder()
    .expireAfterAccess(5, TimeUnit.MINUTES)
    .build(key -> createExpensiveGraph(key));
LoadingCache<Key, Graph> graphs = Caffeine.newBuilder()
    .expireAfterWrite(10, TimeUnit.MINUTES)
    .build(key -> createExpensiveGraph(key));

// Evict based on a varying expiration policy
LoadingCache<Key, Graph> graphs = Caffeine.newBuilder()
    .expireAfter(new Expiry<Key, Graph>() {
      public long expireAfterCreate(Key key, Graph graph, long currentTime) {
        // Use wall clock time, rather than nanotime, if from an external resource
        long seconds = graph.creationDate().plusHours(5)
            .minus(System.currentTimeMillis(), MILLIS)
            .toEpochSecond();
        return TimeUnit.SECONDS.toNanos(seconds);
      }
      public long expireAfterUpdate(Key key, Graph graph,
          long currentTime, long currentDuration) {
        return currentDuration;
      }
      public long expireAfterRead(Key key, Graph graph,
          long currentTime, long currentDuration) {
        return currentDuration;
      }
    })
    .build(key -> createExpensiveGraph(key));
```

Caffeine 提供了三种用法：

- `expireAfterAccess(long, TimeUnit)`: 在一段时间无读写操作时过期，通常在缓存数据和 session绑定，或不活跃过期的场景下使用。
- `expireAfterWrite(long, TimeUnit)`: 写后一段时间过期。主要用于缓存数据在一段时间变旧的场景。
- `expireAfter(Expiry)`: 在指定时间后使条目过期。如果过期时间由外部资源决定，则这个方案可能是最合适的。

过期在写和偶尔的读操作中周期性的维护。调度和触发过期时间复杂度为 摊还常量。

为了快速过期，而不是使用缓存活动来触发周期性维护，可以使用 `Scheduler` 接口和 `Caffeine.scheduler(Scheduler)` 方法来指定一个调度线程。Java 9+ 的用户可以优先选择 `Scheduler.systemScheduler()` 来利用系统范围内的调度线程。

测试可以使用 `Ticker` 接口和 `Caffeine.ticker(Ticker)` 方法来指定一个时间源，而不是等待系统时钟。为了这个目的，Guava 的测试库提供了方便的 `FakeTicker`。

#### 基于引用的驱逐

```java
// Evict when neither the key nor value are strongly reachable
LoadingCache<Key, Graph> graphs = Caffeine.newBuilder()
    .weakKeys()
    .weakValues()
    .build(key -> createExpensiveGraph(key));

// Evict when the garbage collector needs to free memory
LoadingCache<Key, Graph> graphs = Caffeine.newBuilder()
    .softValues()
    .build(key -> createExpensiveGraph(key));
```

Caffeine 允许配置缓存来允许 GC 回收条目，通过使用 key - 弱引用，value - 软/弱引用。`AsyncCache` 不支持软引用和弱引用。

- `CacheBuilder.weakKeys()` 使用弱引用存储 Key。这允许在key 没有强引用或软引用指向时，GC 回收条目。因为垃圾回收仅仅依赖于引用相等性，这导致整个缓存使用 `==` 来判断 Key 是否相等，而不是 `equals`，这个有点危险！！！。
- `CacheBuilder.weakValues()` 使用弱引用存储 Value。这允许在 Value 没有强引用或软引用指向时，GC 回收条目。因为垃圾回收仅仅依赖于引用相等性，这导致整个缓存使用 `==` 来判断 Value 是否相等。
- `CacheBuilder.softValues()` 使用软引用存储 Value。为了释放内存资源，GC 时，软引用对象使用全局 LRU 的方式来回收，因为使用软引用会对性能有影响，所以我们通常推荐使用可预测的最大缓存大小的配置来替代这个配置。使用软引用将导致 Value 使用 `==` 来判断相等性。

### 移除

术语：

- 驱逐：通过过期策略移除
- 失效：调用者手动移除
- 移除：是失效或者驱逐的最终结果

#### 显式删除

在任何时候，你都可以显式的废除缓存条目，而不是等待条目被驱逐。

- 对单个 key，`Cache.invalidate(key)`
- 批量，`Cache.invalidateAll(keys)`
- 所有实体，`Cache.invalidateAll()`

#### 移除监听器

```java
Cache<Key, Graph> graphs = Caffeine.newBuilder()
    .removalListener((Key key, Graph graph, RemovalCause cause) ->
        System.out.printf("Key %s was removed (%s)%n", key, cause))
    .build();
```

你可以使用 `CacheBuilder.removalListener(RemovalListener)` 为你的缓存指定移除监听器，在实体被移除时执行一些操作。在条目被移除时，会将 `RemovalNotification` 传递给移除事件监听器，里面会带有 `RemovalCause`，key 和 Value。

移除监听器操作使用 `[Executor][Executor]` 异步执行。默认执行器是 [`ForkJoinPool`][ForkJoinPool]。可以通过 `Caffeine.executor(Executor)` 自定义。当移除时操作必须同步时，需要使用 `CacheWriter`。

`RemovalListener` 中的异常在输出到日志后被吞掉。

### 刷新

```java
LoadingCache<Key, Graph> graphs = Caffeine.newBuilder()
    .maximumSize(10_000)
    .refreshAfterWrite(1, TimeUnit.MINUTES)
    .build(key -> createExpensiveGraph(key));
```

刷新与驱逐并不完全相同。如 `LoadingCache.refresh` 会异步加载该键的新值。与驱逐相反，旧密钥（如果有的话）在刷新密钥时仍会返回，这迫使查询要等到新值被重新加载。

与 `expireAfterWrite` 相比，`refreshAfterWrite` 将使键在指定的持续时间后有资格进行刷新，但是仅在查询条目时才真正启动刷新。因此，可以在同一缓存上同时指定 `refreshAfterWrite` 和 `expireAfterWrite`，这样，只要条目有资格进行刷新，就不会盲目地重置条目的过期计时器。如果条目符合刷新资格后仍未查询，则允许该条目过期。

`CacheLoader` 可以通过重写 `CacheLoader.reload` 来指定要在刷新时的行为，该行为允许您在计算新值时使用旧值。

刷新操作是使用异步执行的。默认是 [`ForkJoinPool`][ForkJoinPool]，可以通过 `Caffeine.executor` 重写。

如果刷新时抛出异常，则保留旧值，记录日志（使用Logger）然后吞下该异常。

### Writer

```java
LoadingCache<Key, Graph> graphs = Caffeine.newBuilder()
  .writer(new CacheWriter<Key, Graph>() {
    @Override public void write(Key key, Graph graph) {
      // write to storage or secondary cache
    }
    @Override public void delete(Key key, Graph graph, RemovalCause cause) {
      // delete from storage or secondary cache
    }
  })
  .build(key -> createExpensiveGraph(key));
```

`CacheWriter` 和 `CacheLoader` 集合，代理所有的读写操作，允许缓存作为底层资源的门面。

`CacheWriter` 更新缓存和外部资源是原子的，这意味着，在写操作未完成前，这个key对应的新的写操作将会被阻塞，读操作会读到旧值，如果写失败了，保留旧值，并向缓存调用方抛出异常。

条目的创建、修改、移除 都会调用 `CacheWriter`。加载 `LoadingCache.get`,重载 `LoadingCache.refresh`，计算 `Map.computeIfPresent` 操作不会调用。

`CacheWriter` 不能和 `weak keys` 或 `AsyncLoadingCache` 结合使用。

#### 可能的用例

`CacheWriter` 对类似于外部资源需要获取指定 key 内容变更顺序的复杂工作流来说是一个拓展点。Caffeine 支持下面的几种用例，但不是原生支持。

##### 写模式

`CacheWriter` 可以用来实现 `write-through` 或 `write-back` 模式的缓存。

`write-through` 模式下操作是同步的，缓存只有在成功更新外部资源后才会被更新。这防止了外部资源和缓存作为原子操作被更新时的竞争条件。

`write-back` 模式是在更新缓存成功后异步更新外部资源，这种模式牺牲了一致性来提高写的吞吐量，比如异步写失败时的缓存不合法问题。这种方式在延迟一定时间写、限制写操作频率或者批量写时比较有用。

`write-back` 模式可以考虑支持以下的特性：

- 批处理和合并操作
- 操作延迟一定时间窗口
- 如果配置了定时刷新，请在定期刷新之前执行批处理
  > `Performing a batch prior to a periodic flush if it exceeds a threshold size`
- 如果修改还没刷新到外部资源，加载时建议优先从修改操作缓存池中加载
- 按照外部资源的特性，处理重试、速率限制和并发度。

可以参考使用 [`RxJava`][RxJava] 写的一个简单样例 [`write-behind-rxjava`](https://github.com/ben-manes/caffeine/tree/master/examples/write-behind-rxjava)

##### 分层

`CacheWriter` 可以用来整合多个缓存层。

分层缓存从记录系统支持的外部缓存加载和写入。这允许我们优先使用一个小的高速缓存，失败后降级到相对慢些的容量更大的缓存。通常的层级有 堆外、本地磁盘、远程缓存。

`victim cache` 是分层缓存的一个变种，它表示被驱逐的条目会写入到二级缓存中。 `delete(K, V, RemovalCause)` 允许检查条目的移除原因并作相应的操作。

##### 同步监听器

`CacheWriter` 可以用来发布同步监听器。

同步监听器按照指定key上的缓存操作的发生顺序来接受事件通知。监听器可以阻塞缓存操作或者将事件入队列来异步处理。这个类型的监听器一般用来复制或者构造一个分布式缓存。

### 统计

```java
Cache<Key, Graph> graphs = Caffeine.newBuilder()
    .maximumSize(10_000)
    .recordStats()
    .build();
```

我们可以通过使用 `Caffeine.recordStats()` 来打开统计信息收集。 `Cache.stats()` 方法返回 `CacheStats`，其中提供了一些统计指标，例如：

- `hitRate()` 命中率。
- `evictionCount()` 缓存驱逐数。
- `averageLoadPenalty()` 新值平均载入延迟。

这些统计信息在缓存调优中至关重要，我们建议在性能比较重要的应用程序中留意这些统计信息。

缓存指标可以和基于推/拉模式的报告系统进行整合。基于拉模式的可以定时调用 `Cache.stats()` 来记录最新的快照。基于推模式的可以自定义一个StatsCounter，在缓存操作执行时直接推送指标。

一个使用 [Dropwizard Metrics](http://metrics.dropwizard.io/) 的简单例子 [stats-metrics](https://github.com/ben-manes/caffeine/tree/master/examples/stats-metrics)

如果使用 [Prometheus](https://prometheus.io/) 的话，可以尝试 [simpleclient-caffeine](https://github.com/prometheus/client_java#caches)

不好选择的话，也可以使用 [Micrometer](http://micrometer.io/) 来整合。

### 清理

默认情况下，Caffeine 不会再条目过期后立即做清理、驱逐操作。Caffeine 在写操作后做少量维护工作，如果写操作比较少的话，偶尔会在读操作后做。如果你的缓存具备高吞吐量的特性，那边你不需要担心维护的问题。如果你的缓存读多写少，你可能会希望开一个外部线程来主动触发清理操作。

```java
LoadingCache<Key, Graph> graphs = Caffeine.newBuilder()
    .scheduler(Scheduler.systemScheduler())
    .expireAfterWrite(10, TimeUnit.MINUTES)
    .build(key -> createExpensiveGraph(key));
```

通过提供一个 `Scheduler` 来主动促进过期条目的移除。以固定频率调度过期事件以利用批量处理并在短期内最大程度地减少执行。调度是尽力而为的，并不能保证具体何时删除过期的条目。 `Java 9+` 的用户可以优先选择 `Scheduler.systemScheduler()` 来利用系统范围内的调度线程。

```java
Cache<Key, Graph> graphs = Caffeine.newBuilder().weakValues().build();
Cleaner cleaner = Cleaner.create();

cleaner.register(graph, graphs::cleanUp);
graphs.put(key, graph);
```

`Java 9+` 的用户可以使用 [`Cleaner`][Cleaner] 来促进基于引用的条目的删除（如果使用了 `weakKeys`、`weakValues` 或者 `softValues`）。简单的将缓存注册到 `Cleaner` 来触发维护操作。

### 策略

缓存支持的策略在构造时确定。这个配置可以在运行时检查和调整。策略使用 [`Optional`][Optional] 来表示缓存是否支持策略。

#### 基于大小

过期策略接口 `Eviction`

```java
cache.policy().eviction().ifPresent(eviction -> {
  eviction.setMaximum(2 * eviction.getMaximum());
});
```

如果缓存受最大权重限制，则可以使用 `weightedSize()` 获得当前权重。这与 `Cache.estimatedSize()` 不同，后者报告存在的条目数。

可以从 `getMaximum()` 中读取最大容量或权重，并使用 `setMaximum(long)` 进行调整。不在新阈值之内的条目将会被驱逐。

如果需要最可能保留或者驱逐的条目列表快照，可以使用 `hottest(int)` 和 `coldest(int)` 来获取。

#### 基于时间

过期策略接口 `Expiration` 或者 `VarExpiration`

```java
cache.policy().expireAfterAccess().ifPresent(expiration -> ...);
cache.policy().expireAfterWrite().ifPresent(expiration -> ...);
cache.policy().expireVariably().ifPresent(expiration -> ...);
cache.policy().refreshAfterWrite().ifPresent(expiration -> ...);
```

`ageOf(key, TimeUnit)` 提供了条目的空闲时间在 `expireAfterAccess`、`expireAfterWrite`、`refreshAfterWrite` 上的视图。

最大有效时间可以通过 `Expiration#getExpiresAfter(TimeUnit)` 获取，通过 `Expiration#setExpiresAfter(long, TimeUnit)` 来调整。

如果需要最可能保留或者过期的条目列表快照，可以使用 `youngest(int)` 和 `oldest(int)` 来获取。

### 测试

```java
FakeTicker ticker = new FakeTicker(); // Guava's testlib
Cache<Key, Graph> cache = Caffeine.newBuilder()
    .expireAfterWrite(10, TimeUnit.MINUTES)
    .executor(Runnable::run)
    .ticker(ticker::read)
    .maximumSize(10)
    .build();

cache.put(key, graph);
ticker.advance(30, TimeUnit.MINUTES)
assertThat(cache.getIfPresent(key), is(nullValue()));
```

测试基于时间的驱逐并不需要等待系统时钟。使用 `Ticker` 和 `Caffeine.ticker(Ticker)` 方法可以指定时间源来替代系统时钟。`Guava 的 testlib` 提供了方便的 `FakeTicker`。可以主动调用 `Cache.cleanUp()` 来触发过期条目的移除。

Caffeine 将 周期维护、移除通知，异步计算使用 `Executor` 来执行。这一定程度上降低了调用者执行成本，默认使用 [`ForkJoinPool.commonPool()`][ForkJoinPool]，可以使用 `Caffeine.executor(Executor)` 方法来指定执行这些操作的线程池。

我们推荐使用 [Awaitility](https://github.com/jayway/awaitility) 来做多线程测试。

### FAQ

#### 固定条目

固定条目无法被驱逐策略驱逐。这个在条目是一个有状态的资源时比较有用，比如锁，只能被使用它的 Client 用完后丢弃。在这些常经理条目的驱逐和重计算会造成资源泄露。

在基于容量的驱逐策略中，可以调整条目的权重到 0 来将该条目排除。这个条目不会计入总容量并且驱逐策略驱逐时会跳过这个条目。自定义的 `Weigher` 需要能够判断一个条目是否是固定条目。

在条目写入缓存时会计算权重和过期时间。可以使用 `cache.asMap().compute` 来添加/删除固定条目。

#### 递归计算

在原子操作内部执行的加载，计算或回调可能不会写入缓存。 ConcurrentHashMap 递归写入，可能导致活动锁（Java 8）或 `IllegalStateException`（Java 9）。

一种解决方法是异步执行计算，例如通过使用 `AsyncLoadingCache`。 在这种情况下，已经建立了映射，值是 `CompletableFuture`，并且在缓存的原子范围之外执行计算。 如果发生了无序的依赖关系链，或者执行程序耗尽了其线程池，这仍然可能会死锁。 为了最大程度地减少这种情况，应在 `Caffeine.executor` 上设置一个 `Executors.newCachedThreadPool()`。

#### 写争夺

一种场景是正在进行计算的条目数和缓存曾经维护的最大条目数相比，接近或者更多。当正在计算的条目数接近 `ConcurrentHashMap` 总容量时，map 的 resize 会在计算完成后操作。

可能发生在缓存预热时（相似但不完全是），在小缓存中可能会更常见一些，正在计算的条目数和缓存容量相近。可以考虑增大初始容量或使用异步缓存。

`ConcurrentHashMap` 对于锁争夺的描述

> Lock contention probability for two threads accessing distinct elements is roughly 1 / (8 * #elements) under random hashes.

## 性能

### 设计

#### 访问操作有序队列

// todo 当前部分现有能力无法理解，后面补充

双链表排序哈希表中的所有条目。通过在哈希表中找到条目，然后操纵其相邻元素，可以在 O(1) 时间内对条目进行操作。

当条目创建、更新、访问时，条目的访问顺序会被调整。最近使用最少的条目排在头部，最近最多的排在尾部。这为基于大小的驱逐（maximumSize）和基于时间驱逐（expireAfterAccess）提供了支持。挑战在于，每次访问都需要对该列表进行更改，从而该列表本身无法实现高效并发。

#### 写操作有序队列

// todo 当前部分现有能力无法理解，后面补充

写入顺序用条目的创建或更新时间来定义。与访问操作队列类似，写顺序队列在O（1）时间中操作。此队列用于写操作后固定时间内过期（expireAfterWrite）

#### 分层的计时器轮

// todo 当前部分现有能力无法理解，后面补充

一个具有时间意识的优先级队列，该队列使用哈希和双向链接列表在 O(1) 时间内执行操作。此队列用于到期时间不固定的场景 expireAfter(Expiry)。

[ton97-timing-wheels](http://www.cs.columbia.edu/~nahum/w6998/papers/ton97-timing-wheels.pdf)

#### 读缓冲区

// todo 当前部分现有能力无法理解，后面补充

#### 写缓冲区

// todo 当前部分现有能力无法理解，后面补充

#### 锁开销均摊

传统的缓存会锁定每个操作以执行少量工作，而 Caffeine 使用批处理将成本分散到许多线程中。这将对锁竞争的成本进行均摊。尽管如果任务被拒绝或使用了调用者运行策略，维护也可以委托用户线程执行。

批处理的一个优点是，由于锁的排他性，缓冲区仅在给定的时间被单个线程消费。 这允许使用更有效的基于多生产者/单消费者的缓冲区实现。 通过利用CPU缓存，它也可以更好地与硬件特性保持一致。

#### 条目状态转换

当缓存不受排他锁保护时，一个挑战是操作可能会以错误的顺序记录和重放。 由于竞争的原因，创建-读取-更新-删除操作可能无法以相同顺序存储在缓冲区中。 这样做将需要粗粒度的锁，从而降低性能。

与并发数据结构中的典型情况一样，咖啡因使用原子状态转换解决了这一难题。 条目是活跃的，已退休的或已死的。 活跃状态意味着它在哈希表和访问/写入队列中都存在。 从哈希表中删除条目时，该条目被标记为已退休，需要从队列中删除。 发生这种情况时，该条目将被视为已失效并且可以进行垃圾回收。

#### 代码生成

有许多不同的配置选项，仅当启用某些功能子集时才需要大多数字段。 如果默认情况下所有字段都存在，则可能会增加缓存和每个条目的开销，从而造成浪费。 通过代码生成，可以减少运行时内存的开销，但需要更大的磁盘二进制文件。

该技术有用算法优化的潜力。 也许在构造缓存时，用户可以指定最适合其使用的特性。 移动应用程序可能更喜欢并发率更高，而服务器应用程序可能需要更高的命中率，但会消耗内存。 意图可能会驱动算法的选择，而不是试图为所有使用找到最佳的平衡。

#### 被装饰的哈希表

### 效率

// todo

### 基准测试

见 [Benchmarks](https://github.com/ben-manes/caffeine/wiki/Benchmarks)

### 内存开销

见 [Memory overhead](https://github.com/ben-manes/caffeine/wiki/Memory-overhead)

## 参考

- [ben-manes/caffeine][ben-manes/caffeine]

[ben-manes/caffeine]: https://github.com/ben-manes/caffeine/wiki
[ConcurrentMap]:https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ConcurrentMap.html
[Executor]:https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/Executor.html
[CompletableFuture]:https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html
[ForkJoinPool]:https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ForkJoinPool.html
[RxJava]:https://github.com/ReactiveX/RxJava
[Cleaner]:https://docs.oracle.com/javase/9/docs/api/java/lang/ref/Cleaner.html
[Optional]:https://docs.oracle.com/javase/8/docs/api/java/util/Optional.html
