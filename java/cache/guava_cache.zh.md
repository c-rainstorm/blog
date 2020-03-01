# Guava Cache Wiki 中文版

搞完 JSR107，本来想搞 Caffeine 的，但是 Caffeine 灵感来自 Guava Cache，为了便于理解 Caffeine 的设计，先了解一下 Guava Cache。

文档原文来自 [Google Guava - GitHub](https://github.com/google/guava/wiki/CachesExplained)。

## 例子

```java
LoadingCache<Key, Graph> graphs = CacheBuilder.newBuilder()
       .maximumSize(1000)
       .expireAfterWrite(10, TimeUnit.MINUTES)
       .removalListener(MY_LISTENER)
       .build(
           new CacheLoader<Key, Graph>() {
             @Override
             public Graph load(Key key) throws AnyException {
               return createExpensiveGraph(key);
             }
           });
```

## 适用性

缓存拥有很广泛的应用场景。例如，当值的计算或检索成本很高，并且在特定输入上将需要多次使用该值时您应该考虑使用缓存。

`Cache` 类似于 `ConcurrentMap`，但并不完全相同。最根本的区别是 `ConcurrentMap` 会保留所有添加到其中的元素，直到将其明确删除为止。另一方面，通常将 `Cache` 配置为自动驱逐条目，以限制其内存占用量。在某些情况下，由于 `LoadingCache` 自动缓存加载，即使不驱逐条目，它也很有用。

Guava Cache 工具适用于以下情况：

- 愿意花费一些内存来提高速度。
- 特定的key有时会多次查询。
- 缓存不需要存储超出RAM容量的数据。 (Guava缓存在应用程序的一次运行中是本地的。它们不将数据存储在文件中或外部服务器上。如果这不满足您的需求，请考虑使用像 Memcached 这样的工具。）

如果这些都适用于您的应用场景，那么 Guava Cache 将很适合您！

如上面的示例代码所示，使用 `CacheBuilder` builder 模式可以获取缓存，自定义缓存这部分很有意思。

注意：如果不需要缓存的功能，则 `ConcurrentHashMap` 的内存使用效率更高，但要用任何已有的 `ConcurrentMap` 复制大多数缓存功能是极其困难或不可能的。

## 缓存加载

在用缓存时，首先应该回答的问题是：是否有一些默认函数使用给定的 Key 来加载或计算对应的 Value。如果答案为是，那么应该使用 `CacheLoader`，如果答案为否，或者你想要覆盖默认的行为，但仍然希望使用原子的 `get-if-absent-compute` 语义，那么应该给 `get()` 调用传递 `Callable`。元素也可以通过 `Cache.put` 直接插入。但是首选的方案依旧是自动缓存加载，因为这样能更容易的保证缓存内容的一致性。


### 从 `CacheLoader`

`LoadingCache` 是 `Cache` + `CacheLoader`，创建 `CacheLoader` 只需要实现 `V load(K key) throws Exception` 方法。

`load()` 定义：

```java
/**
 * Computes or retrieves the value corresponding to {@code key}.
 *
 * @param key the non-null key whose value should be loaded
 * @return the value associated with {@code key}; <b>must not be null</b>
 * @throws Exception if unable to load the result
 * @throws InterruptedException if this method is interrupted. {@code InterruptedException} is
 *     treated like any other {@code Exception} in all respects except that, when it is caught,
 *     the thread's interrupt status is set
 */
public abstract V load(K key) throws Exception;
```

你可以使用下面的代码来创建一个 `LoadingCache`。

```java
LoadingCache<Key, Graph> graphs = CacheBuilder.newBuilder()
       .maximumSize(1000)
       .build(
           new CacheLoader<Key, Graph>() {
             public Graph load(Key key) throws AnyException {
               return createExpensiveGraph(key);
             }
           });

...
try {
  return graphs.get(key);
} catch (ExecutionException e) {
  throw new OtherException(e.getCause());
}
```

查询 `LoadingCache` 的规范方法是使用 `get(K)`方法。 这会返回一个已经缓存的值，或者使用缓存的 `CacheLoader` 自动地将新值加载到缓存中。 由于 `CacheLoader` 可能会引发异常，因此 `LoadingCache.get(K)` 会引发 `ExecutionException`。 (如果缓存加载器抛出未检查的异常，则 `get(K)` 会抛出一个 `UncheckedExecutionException` 对其进行包装。）您还可以选择使用 `getUnchecked(K)`，它将所有异常都包装在 `UncheckedExecutionException`中，但是如果底层的 `CacheLoader` 能正常的抛出检测异常，这通常会引发未知的行为。

```java
LoadingCache<Key, Graph> graphs = CacheBuilder.newBuilder()
       .expireAfterAccess(10, TimeUnit.MINUTES)
       .build(
           new CacheLoader<Key, Graph>() {
             public Graph load(Key key) { // no checked exception
               return createExpensiveGraph(key);
             }
           });

...
return graphs.getUnchecked(key);
```

批量查询可以使用 `getAll(Iterable<? extends K>)`。默认情况下，`getAll` 对并未在缓存中的每个 key 单独调用 `CacheLoader.load` 来加载。

当批量查询比单独查询更有效时，可以覆写 `CacheLoader.loadAll`。`getAll` 的性能会有相应提升。

> 提醒：看了看代码，这块写的有点问题，应该是覆写 `getAll`

请注意，您可以编写一个 `CacheLoader.loadAll` 实现，该实现加载未明确要求的键的值。 例如，如果计算某个集合中任何键的值为您提供了该组中所有键的值，那么 `loadAll` 可能会同时加载该组中的其余键。

### 从 `Callable`

所有的 Guava Cache 均支持 `get(K，Callable <V>)` 方法。 此方法返回与缓存中的键关联的值，或从指定的 `Callable` 中计算出该值并将其添加到缓存中。 在加载完成之前，不会修改与此缓存关联的可观察状态。 此方法为常规的“如果已缓存，则返回；否则创建，缓存并返回”模式提供了简单的替代方法。

```java
Cache<Key, Value> cache = CacheBuilder.newBuilder()
    .maximumSize(1000)
    .build(); // look Ma, no CacheLoader
...
try {
  // If the key wasn't in the "easy to compute" group, we need to
  // do things the hard way.
  cache.get(key, new Callable<Value>() {
    @Override
    public Value call() throws AnyException {
      return doThingsTheHardWay(key);
    }
  });
} catch (ExecutionException e) {
  throw new OtherException(e.getCause());
}
```

### 直接插入

值可以通过 `cache.put(key,value)` 的方式直接插入缓存。 这将覆盖 `Cache` 中指定键的先前条目。也可以使用 `Cache.asMap()` 视图公开的任何 `ConcurrentMap` 方法对缓存进行更改。 请注意，`asMap` 视图上的任何方法都不会导致条目自动加载到缓存中。 因此在使用 `CacheLoader` 或 `Callable` 来加载值的缓存中，与 `Cache.asMap().putIfAbsent` 相比，应始终首选 `Cache.get(K，Callable<V>)`。

## 驱逐

现实是，我们几乎肯定不会有足够内存来缓存所有需要的数据。你必须决定在内存不足时怎么样判断哪些条目是可以被驱逐的。Guava Cache 提供了三种基本的驱逐类型: 基于大小的驱逐，基于时间的驱逐，基于引用的驱逐。

### 基于大小的驱逐

如果你的缓存不应该超过特定的大小，那么只需要使用 `CacheBuilder.maximumSize(long)`。缓存将尝试驱逐最近最少使用的条目。需要注意的是，缓存可能在达到限制之前就已经开始驱逐条目了，同通常来说这发生在接近该限制时。

或者，不同的缓存条目有不同的权重(例如条目的内存占用量）。此时可以使用 `CacheBuilder.weigher(Weigher)` 指定权重函数，用 `CacheBuilder.maximumWeight(long)` 来指定最大权重。需要注意的是可能在达到限制之前就开始驱逐条目了，而且条目权重会在创建时进行计算。

```java
LoadingCache<Key, Graph> graphs = CacheBuilder.newBuilder()
       .maximumWeight(100000)
       .weigher(new Weigher<Key, Graph>() {
          public int weigh(Key k, Graph g) {
            return g.vertices().size();
          }
        })
       .build(
           new CacheLoader<Key, Graph>() {
             public Graph load(Key key) { // no checked exception
               return createExpensiveGraph(key);
             }
           });
```

### 基于时间的驱逐

`CacheBuilder` 提供了两种方式：

- `expireAfterAccess(long, TimeUnit)`，只过期在最近一定时间内未进行读写操作的条目。需要注意的是条目驱逐的顺序和 基于大小的驱逐相似。
- `expireAfterWrite(long, TimeUnit)`，过期在最近一定时间内没有写操作的条目。

基于时间的过期在写入过程中定时维护，偶尔在读操作中维护，下面会讨论这部分。

#### 测试基于时间的驱逐

测试定时驱逐并不一定会很痛苦，实际上也不需要花两秒钟来测试两秒钟的到期时间。 使用 `Ticker` 接口和 `CacheBuilder.ticker(Ticker)`方法可以在缓存生成器中指定时间源，而不必等待系统时钟。

### 基于引用的驱逐

【这个对业务来说有点危险，慎用】
【这个对业务来说有点危险，慎用】
【这个对业务来说有点危险，慎用】

Guava Cache 支持将 Cache 配置为允许 GC 回收条目，通过将 Key 配置为弱引用，Value 配置为 软引用或弱引用。

- `CacheBuilder.weakKeys()` 使用弱引用存储 Key。这允许在key 没有强引用或软引用指向时，GC 回收条目。因为垃圾回收仅仅依赖于引用相等性，这导致整个缓存使用 `==` 来判断 Key 是否相等，而不是 `equals`，这个有点危险！！！。

```java
@Test
public void guavaEqualTest() {
    Cache<String, String> cache = CacheBuilder.newBuilder()
            .weakKeys()
            .build();

    String str1 = "hello";
    String str2 = new String(str1);

    assert str1 != str2;             // pass

    cache.put(str1, str1);
    String value = cache.getIfPresent(str2);
    assert Objects.isNull(value);    // pass
}
```

- `CacheBuilder.weakValues()` 使用弱引用存储 Value。这允许在 Value 没有强引用或软引用指向时，GC 回收条目。因为垃圾回收仅仅依赖于引用相等性，这导致整个缓存使用 `==` 来判断 Value 是否相等。
- `CacheBuilder.softValues()` 使用软引用存储 Value。为了释放内存资源，GC 时，软引用对象使用全局 LRU 的方式来回收，因为使用软引用会对性能有影响，所以我们通常推荐使用可预测的最大缓存大小的配置来替代这个配置。使用软引用将导致 Value 使用 `==` 来判断相等性。

### 显式删除

在任何时候，你都可以显式的废除缓存条目，而不是等待条目被驱逐。

- 对单个 key，`Cache.invalidate(key)`
- 批量，`Cache.invalidateAll(keys)`
- 所有实体，`Cache.invalidateAll()`

### 移除监听器

你可以使用 `CacheBuilder.removalListener(RemovalListener)` 为你的缓存指定移除监听器，在实体被移除时执行一些操作。在条目被移除时，会将 `RemovalNotification` 传递给移除事件监听器，里面会带有 `RemovalCause`，key 和 Value。

`RemovalListener` 中的异常在输出到日志后被吞掉。

```java
CacheLoader<Key, DatabaseConnection> loader = new CacheLoader<Key, DatabaseConnection> () {
  public DatabaseConnection load(Key key) throws Exception {
    return openConnection(key);
  }
};
RemovalListener<Key, DatabaseConnection> removalListener = new RemovalListener<Key, DatabaseConnection>() {
  public void onRemoval(RemovalNotification<Key, DatabaseConnection> removal) {
    DatabaseConnection conn = removal.getValue();
    conn.close(); // tear down properly
  }
};

return CacheBuilder.newBuilder()
  .expireAfterWrite(2, TimeUnit.MINUTES)
  .removalListener(removalListener)
  .build(loader);
```

警告：移除监听器默认同步执行，并且由于缓存维护通常是在常规缓存操作期间执行的，因此昂贵的删移除侦听器会降低正常的缓存功能的性能！ 如果您拥有昂贵的删除侦听器，请使用 `RemovalListeners.asynchronous(RemovalListener，Executor)` 装饰一个 `RemovalListener` 以便异步运行。

### 清理操作什么时候做

使用 `CacheBuilder` 构建的缓存不会“自动”或在值过期后立即执行清除和逐出值，或执行任何类似的操作。取而代之的是，如果写操作很少，则在写操作期间或偶尔的读操作期间，它将执行少量维护。

这样做的原因如下：如果我们要连续执行 `Cache` 维护，则需要创建一个线程，并且该线程的操作将与用户操作争夺共享锁。此外，某些环境限制了线程的创建，这将使 `CacheBuilder` 在该环境中无法使用。

相反，我们会将选择权交给您。如果您的缓存是高吞吐量的，那么您不必担心执行缓存维护以清理过期的条目等。如果您的缓存确实很少写入，并且您不想清理来阻止缓存读取，这时您可能希望创建自己的维护线程，该线程定期调用 `Cache.cleanUp()`。

如果要为很少有写入的缓存安排定期的缓存维护，只需使用 `ScheduledExecutorService` 调度维护。

### 刷新

刷新与驱逐并不完全相同。 如 `LoadingCache.refresh(K)` 中所指定，刷新键可能会异步加载该键的新值。 与驱逐相反，旧值(如果有的话）在刷新时仍会返回，这迫使查询要等到重新加载该值。

如果刷新时引发异常，则保留旧值，记录日志并吞下异常。

`CacheLoader` 可以通过重写 `CacheLoader.reload(K，V)` 来指定要在刷新时使用的行为，这使您可以在计算新值时使用旧值。

```java
// Some keys don't need refreshing, and we want refreshes to be done asynchronously.
LoadingCache<Key, Graph> graphs = CacheBuilder.newBuilder()
       .maximumSize(1000)
       .refreshAfterWrite(1, TimeUnit.MINUTES)
       .build(
           new CacheLoader<Key, Graph>() {
             public Graph load(Key key) { // no checked exception
               return getGraphFromDatabase(key);
             }

             public ListenableFuture<Graph> reload(final Key key, Graph prevGraph) {
               if (neverNeedsRefresh(key)) {
                 return Futures.immediateFuture(prevGraph);
               } else {
                 // asynchronous!
                 ListenableFutureTask<Graph> task = ListenableFutureTask.create(new Callable<Graph>() {
                   public Graph call() {
                     return getGraphFromDatabase(key);
                   }
                 });
                 executor.execute(task);
                 return task;
               }
             }
           });
```

可以使用 `CacheBuilder.refreshAfterWrite(long，TimeUnit)` 将自动定时刷新添加到缓存中。 与 `expireAfterWrite` 相比，`refreshAfterWrite` 将使键在指定的持续时间后有资格进行刷新，但是仅在查询条目时才真正启动刷新。 (如果将 `CacheLoader.reload` 实现为异步，则刷新不会降低查询的速度。）因此，例如，您可以在同一缓存上同时指定 `refreshAfterWrite` 和 `expireAfterWrite`，只要条目符合刷新资格，条目的到期计时器就不会盲目重置，因此，如果在符合刷新资格后仍未查询该条目，则允许该条目过期。

## 特性

### 统计

通过使用 `CacheBuilder.recordStats()`，可以打开 Guava Cache 的统计信息收集。`Cache.stats()` 方法返回 `CacheStats` 对象，该对象提供统计信息，例如

- `hitRate`，返回命中率
- `averageLoadPenalty()`，加载新值所花费的平均时间，以纳秒为单位
- `evictionCount()`，缓存驱逐的数量

以及其他更多统计信息。 这些统计信息在缓存调整中至关重要，我们建议在性能至关重要的应用程序中留意这些统计信息。

### `asMap`

您可以使用其 `asMap` 视图将任何缓存作为 `ConcurrentMap` 查看，但是 `asMap` 视图如何与 `Cache` 交互需要简单解释一下。

`cache.asMap()` 包含当前在缓存中加载的所有条目。 因此，`cache.asMap().keySet()` 包含所有当前加载的键。
`asMap().get(key)`本质上等效于 `cache.getIfPresent(key)`，并且从不加载值。 这和 Map 一样。

所有缓存读取和写入操作(包括 `Cache.asMap().get(Object)` 和 `Cache.asMap().put(K，V))` 都会重置访问时间，但`containsKey(Object)` 和 `Cache.asMap()` 不会。 因此，遍历 `cache.asMap().entrySet()` 不会重置您检索的条目的访问时间。

### 中断

加载方法（如get）永远不会抛出 `InterruptedException`。我们本可以设计这些方法来支持 `InterruptedException`，但是我们的支持将是不完整的，迫使所有用户付出代价，但只有部分受益。有关详细信息，请继续阅读。

查询未缓存值的get调用可分为两大类：加载值的调用和等待另一个线程正在进行的加载的调用。两者在支持中断的能力上有所不同。最简单的情况是等待另一个线程的正在进行的加载：在这里我们可以进入一个可中断的等待。难中断的是我们自己加载值的情况。在这里，我们受用户提供的 `CacheLoader` 的支配。如果碰巧支持中断，我们可以支持中断。如果没有，我们不能。

那么，为什么在提供 `CacheLoader` 支持时不支持中断呢？从某种意义上说，我们做到了（请参见下文）：如果 `CacheLoader` 抛出`InterruptedException`，则对键的所有get调用将立即返回（与其他任何异常一样）。另外，get将恢复加载线程中的中断位。令人惊讶的部分是`InterruptedException` 包装在 `ExecutionException` 中。

原则上，我们可以为您解开此异常。但是，这将强制所有 `LoadingCache` 用户处理 `InterruptedException`，即使大多数 `CacheLoader`实现从不抛出该异常。所有非加载线程的等待仍可能被中断时，也许这依旧是值得的。但是许多缓存仅在单个线程中使用。他们的用户仍然必须捕获不可能的 `InterruptedException`。而且，即使是那些在线程间共享缓存的用户，有时也只能根据哪个线程首先发出请求来中断其get调用。

在此决策中，我们的指导原则是使缓存的行为就像所有值都已加载到调用线程中一样。这个原则我们可方便的将缓存引入每次调用都重新计算值的代码中。而且，如果旧代码不可中断，那么可以选择新代码。

我们在某种意义上支持中断。我们还有另一种感觉，那就是使 `LoadingCache` 成为泄漏的抽象。如果加载线程被中断，我们会像对待其他异常一样对待它。在许多情况下都可以，但是当多个get调用正在等待该值时，这不是正确的选择。尽管恰好正在计算该值的操作被中断，但其他需要该值的操作可能不需要中断。然而，所有这些调用者都收到 `InterruptedException`（包装在 `ExecutionException` 中）。正确的行为是让其余线程之一重试加载。我们为此提交了一个bug。但是，修复可能会有风险。除了解决问题之外，我们还可以在拟议的 `AsyncLoadingCache` 中投入更多精力，该方法将返回具有正确中断行为的 `Future` 对象。
