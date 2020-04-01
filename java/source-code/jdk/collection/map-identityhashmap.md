# `IdentityHashMap<K,V>`

线性探测实现，key 为引用，判断 key 相等使用 `==` 而不是 `equals`。

## 容量

```java
/**
    * The initial capacity used by the no-args constructor.
    * MUST be a power of two.  The value 32 corresponds to the
    * (specified) expected maximum size of 21, given a load factor
    * of 2/3.
    */
private static final int DEFAULT_CAPACITY = 32;

/**
    * The minimum capacity, used if a lower value is implicitly specified
    * by either of the constructors with arguments.  The value 4 corresponds
    * to an expected maximum size of 2, given a load factor of 2/3.
    * MUST be a power of two.
    */
private static final int MINIMUM_CAPACITY = 4;

/**
    * The maximum capacity, used if a higher value is implicitly specified
    * by either of the constructors with arguments.
    * MUST be a power of two <= 1<<29.
    *
    * In fact, the map can hold no more than MAXIMUM_CAPACITY-1 items
    * because it has to have at least one slot with the key == null
    * in order to avoid infinite loops in get(), put(), remove()
    */
private static final int MAXIMUM_CAPACITY = 1 << 29;

/**
    * Returns the appropriate capacity for the given expected maximum size.
    * Returns the smallest power of two between MINIMUM_CAPACITY and
    * MAXIMUM_CAPACITY, inclusive, that is greater than (3 *
    * expectedMaxSize)/2, if such a number exists.  Otherwise returns
    * MAXIMUM_CAPACITY.
    */
private static int capacity(int expectedMaxSize) {
    // assert expectedMaxSize >= 0;
    return
        // 期待容量大于最大容量的 1/3 直接使用最大容量
        (expectedMaxSize > MAXIMUM_CAPACITY / 3) ? MAXIMUM_CAPACITY :
        // 期待容量小于等于最小容量的 2/3 直接使用最小容量
        (expectedMaxSize <= 2 * MINIMUM_CAPACITY / 3) ? MINIMUM_CAPACITY :
        // 使用大于期待容量3倍的最小 2^n 次数作为容量
        Integer.highestOneBit(expectedMaxSize + (expectedMaxSize << 1));
}

/**
* Initializes object to be an empty map with the specified initial
* capacity, which is assumed to be a power of two between
* MINIMUM_CAPACITY and MAXIMUM_CAPACITY inclusive.
*/
private void init(int initCapacity) {
   // assert (initCapacity & -initCapacity) == initCapacity; // power of 2
   // assert initCapacity >= MINIMUM_CAPACITY;
   // assert initCapacity <= MAXIMUM_CAPACITY;

   table = new Object[2 * initCapacity];
}
```

## 开源组件拓展

最核心的是 `AnnotationAwareOrderComparator`。

### Spring bean 排序

`DefaultListableBeanFactory` 中 `FactoryAwareOrderSourceProvider` 用于保存 bean instace 到 beanName 的 mapping，spring 框架使用 bean 的引用查找对应的 beanName, 然后查找对应的 `RootBeanDefinition` 然后确定 bean 的优先级。因为 instance 是引用，不适合使用 equals 来判断相等性，因为同一个类的实例，equals 判断相等，但是实际对应两个对象，这种情况转换 beanName -> instance 转为 instance->beanName 的过程会丢 bean，所以只能使用引用来判断相等性，所以需要使用 `IdentityHashMap`。

```java
private OrderComparator.OrderSourceProvider createFactoryAwareOrderSourceProvider(Map<String, ?> beans) {
    IdentityHashMap<Object, String> instancesToBeanNames = new IdentityHashMap<>();
    beans.forEach((beanName, instance) -> instancesToBeanNames.put(instance, beanName));
    return new FactoryAwareOrderSourceProvider(instancesToBeanNames);
}
```

```java
/**
* An {@link org.springframework.core.OrderComparator.OrderSourceProvider} implementation
* that is aware of the bean metadata of the instances to sort.
* <p>Lookup for the method factory of an instance to sort, if any, and let the
* comparator retrieve the {@link org.springframework.core.annotation.Order}
* value defined on it. This essentially allows for the following construct:
*/
private class FactoryAwareOrderSourceProvider implements OrderComparator.OrderSourceProvider {

    private final Map<Object, String> instancesToBeanNames;

    public FactoryAwareOrderSourceProvider(Map<Object, String> instancesToBeanNames) {
        this.instancesToBeanNames = instancesToBeanNames;
    }

    @Override
    @Nullable
    public Object getOrderSource(Object obj) {
        // 根据引用，拿到 beanName，用 beanName 查找对应的 RootBeanDefinition
        RootBeanDefinition beanDefinition = getRootBeanDefinition(this.instancesToBeanNames.get(obj));
        if (beanDefinition == null) {
            return null;
        }
        List<Object> sources = new ArrayList<>(2);
        Method factoryMethod = beanDefinition.getResolvedFactoryMethod();
        if (factoryMethod != null) {
            sources.add(factoryMethod);
        }
        Class<?> targetType = beanDefinition.getTargetType();
        if (targetType != null && targetType != obj.getClass()) {
            sources.add(targetType);
        }
        return sources.toArray();
    }

    @Nullable
    private RootBeanDefinition getRootBeanDefinition(@Nullable String beanName) {
        if (beanName != null && containsBeanDefinition(beanName)) {
            BeanDefinition bd = getMergedBeanDefinition(beanName);
            if (bd instanceof RootBeanDefinition) {
                return (RootBeanDefinition) bd;
            }
        }
        return null;
    }
}
```

### `HttpMessageConverter`

```java
@Configuration
public class HttpMessageConverterConfig {
    @Bean
    @Order(1)
    public HttpMessageConverter<String> messageConverter1() {
        return new StringHttpMessageConverter(StandardCharsets.UTF_8);
    }

    @Bean
    @Order(2)
    public HttpMessageConverter<String> messageConverter2() {
        return new StringHttpMessageConverter(StandardCharsets.UTF_8);
    }

    @Bean
    @Order(3)
    public HttpMessageConverter<String> messageConverter3() {
        return new StringHttpMessageConverter(StandardCharsets.UTF_8);
    }
}
```

### `ApplicationContextInitializer`

```java
@Order(1)
public class ApplicationContextInitializer1 implements ApplicationContextInitializer<ConfigurableApplicationContext> {
    @Override
    public void initialize(ConfigurableApplicationContext applicationContext) {
        System.out.println(this.getClass().getName());
    }
}

@Order(2)
public class ApplicationContextInitializer2 implements ApplicationContextInitializer<ConfigurableApplicationContext> {
    @Override
    public void initialize(ConfigurableApplicationContext applicationContext) {
        System.out.println(this.getClass().getName());
    }
}
```
