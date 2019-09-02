# Spring 容器的生命周期

了解Spring 容器的生命周期，我们可以在容器生命周期的各个阶段执行自定义的操作以拓展应用程序的能力。本片博客的所有源代码，可以在 [boot-init](https://github.com/c-rainstorm/boot-init) 仓库找到。有兴趣的自行查阅。

---

- [Spring 容器的生命周期](#spring-%e5%ae%b9%e5%99%a8%e7%9a%84%e7%94%9f%e5%91%bd%e5%91%a8%e6%9c%9f)
  - [测试日志](#%e6%b5%8b%e8%af%95%e6%97%a5%e5%bf%97)
  - [整体流程图](#%e6%95%b4%e4%bd%93%e6%b5%81%e7%a8%8b%e5%9b%be)
  - [生命周期事件](#%e7%94%9f%e5%91%bd%e5%91%a8%e6%9c%9f%e4%ba%8b%e4%bb%b6)
    - [Spring经典生命周期事件](#spring%e7%bb%8f%e5%85%b8%e7%94%9f%e5%91%bd%e5%91%a8%e6%9c%9f%e4%ba%8b%e4%bb%b6)
    - [Spring boot 生命周期事件](#spring-boot-%e7%94%9f%e5%91%bd%e5%91%a8%e6%9c%9f%e4%ba%8b%e4%bb%b6)
    - [自定义的应用程序事件](#%e8%87%aa%e5%ae%9a%e4%b9%89%e7%9a%84%e5%ba%94%e7%94%a8%e7%a8%8b%e5%ba%8f%e4%ba%8b%e4%bb%b6)
  - [初始化器](#%e5%88%9d%e5%a7%8b%e5%8c%96%e5%99%a8)
  - [JVM `shutdownHook`](#jvm-shutdownhook)

---

## 测试日志

```log

2019-09-02 22:47:43.015  INFO 61026 --- [           main] m.r.b.l.a.ApplicationEventTest           : org.springframework.boot.context.event.ApplicationEnvironmentPreparedEvent
2019-09-02 22:47:43.101  INFO 61026 --- [           main] com.rainstorm.boot.skyLog                : [ApplicationInit][initialize][][] - ; message: 初始化方法调用
2019-09-02 22:47:43.102  INFO 61026 --- [           main] m.r.b.l.a.ApplicationEventTest           : org.springframework.boot.context.event.ApplicationContextInitializedEvent

...

2019-09-02 22:47:43.149  INFO 61026 --- [           main] m.r.b.l.a.ApplicationEventTest           : org.springframework.boot.context.event.ApplicationPreparedEvent

...

2019-09-02 22:47:49.036  INFO 61026 --- [           main] m.r.b.l.a.ApplicationEventTest           : me.rainstorm.boot.lifecycle.application.JacksonConverterConfigDoneEvent

...

2019-09-02 22:47:50.041  INFO 61026 --- [           main] m.r.b.l.a.ApplicationEventTest           : org.springframework.context.event.ContextRefreshedEvent

...

2019-09-02 22:47:50.061  INFO 61026 --- [           main] m.r.b.l.a.ApplicationEventTest           : org.springframework.boot.web.servlet.context.ServletWebServerInitializedEvent

2019-09-02 22:47:50.065  INFO 61026 --- [           main] m.r.b.l.a.ApplicationEventTest           : org.springframework.boot.context.event.ApplicationStartedEvent
2019-09-02 22:47:50.076  INFO 61026 --- [           main] m.r.b.l.a.ApplicationEventTest           : org.springframework.boot.context.event.ApplicationReadyEvent

...

2019-09-02 22:50:41.951  INFO 61026 --- [       Thread-1] m.r.b.l.application.ApplicationInit      : from shutdown hook
2019-09-02 22:50:41.951  INFO 61026 --- [      Thread-18] m.r.b.l.a.ApplicationEventTest           : org.springframework.context.event.ContextClosedEvent
```

## 整体流程图

// todo 周末再搞吧

## 生命周期事件

### Spring经典生命周期事件

![Spring Context event](http://image.rainstorm.vip/blog/spring/spring-application-event.png)

// todo 详情周末搞

### Spring boot 生命周期事件

![Spring boot application event](http://image.rainstorm.vip/blog/spring/spring-boot-application-event.png)

// todo 详情周末搞

### 自定义的应用程序事件

// todo 事件的发布订阅模式

借助 `ApplicationContext` 实现自定义事件发布与监听

监听器事件

```java
@Slf4j
@Component
public class EventPublisherImpl implements ApplicationContextAware, ApplicationEventPublisher {
    private ApplicationContext applicationContext;

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
    }

    @Override
    public void publishEvent(Object event) {
        applicationContext.publishEvent(event);
    }
}
```

自定义事件

```java
public class JacksonConverterConfigDoneEvent extends ApplicationEvent {
    /**
     * Create a new ApplicationEvent.
     *
     * @param source the object on which the event initially occurred (never {@code null})
     */
    public JacksonConverterConfigDoneEvent(Object source) {
        super(source);
    }
}

```

发布事件

```java
@Configuration
public class JacksonConverter {

    @Resource
    private ApplicationEventPublisher eventPublisher;

    @Bean
    public ObjectMapper objectMapper() {
        ObjectMapper objectMapper = new ObjectMapper();

        ....

        eventPublisher.publishEvent(new JacksonConverterConfigDoneEvent(objectMapper));

        return objectMapper;
    }
```

## 初始化器

```java
@Slf4j
public class ApplicationInit implements ApplicationContextInitializer<ConfigurableApplicationContext> {
    private static final String CATEGORY = ApplicationInit.class.getSimpleName();

    @Override
    public void initialize(ConfigurableApplicationContext applicationContext) {
        final String logMethodName = "initialize";

        Runtime.getRuntime().addShutdownHook(new Thread(() -> {
            log.info("from shutdown hook");
        }));

        LogUtil.info(LogBuilder.init(CATEGORY, logMethodName).setMessage("初始化方法调用").build());
    }
}
```

## JVM `shutdownHook`

注册 `ShutdownHook`

```java
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    log.info("from shutdown hook");
}));
```

注意，因为 ShutdownHook 底层使用Map 作为存储结构，且是多线程并发执行，无法精确的指定执行的先后顺序，除非显示的对 Shutdown hook 进行同步处理

```java
static void runHooks() {
    Collection<Thread> threads;
    synchronized(ApplicationShutdownHooks.class) {
        threads = hooks.keySet();
        hooks = null;
    }

    for (Thread hook : threads) {
        hook.start();
    }
    for (Thread hook : threads) {
        while (true) {
            try {
                hook.join();
                break;
            } catch (InterruptedException ignored) {
            }
        }
    }
}
```
