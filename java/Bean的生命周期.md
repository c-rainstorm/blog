# Spring bean 的生命周期

## 概述

![Spring bean 生命周期](http://image.rainstorm.vip/blog/spring/spring-bean-lifecycle.png)

1. 注入属性
2. `Aware` 的 setter 方法注入资源
3. `BeanPostProcessor` 进行初始化前后的处理（`BeanPostProcessor`实现类不能处理 `BeanPostProcessor`实现类）
4. `@PostConstruct`
5. `@PreDestroy`

## 测试类

在 [boot-init](https://github.com/c-rainstorm/boot-init) 项目中, `BeanLifeCycleAllInOne`,`AfterBeanInitial`

```java
@Slf4j
@Component
public class BeanLifeCycleAllInOne implements BeanNameAware, ApplicationContextAware, BeanFactoryAware, EnvironmentAware, ResourceLoaderAware {

    @Override
    public void setBeanName(String name) {
        log.info("BeanNameAware.setBeanName -- {}", name);
    }

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        log.info("ApplicationContextAware.setApplicationContext");
    }


    @Override
    public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
        log.info("BeanFactoryAware.setBeanFactory");
    }

    @Override
    public void setEnvironment(Environment environment) {
        log.info("EnvironmentAware.setEnvironment");
    }


    @Override
    public void setResourceLoader(ResourceLoader resourceLoader) {
        log.info("ResourceLoaderAware.setResourceLoader");
    }

    @PostConstruct
    public void postConstruct() {
        log.info("BeanLifeCycleAllInOne.postConstruct");
    }

    @PreDestroy
    public void preDestroy() {
        log.info("BeanLifeCycleAllInOne.preDestroy");
    }
}
```

```java
@Slf4j
@Component
public class AfterBeanInitial implements BeanPostProcessor {
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        if (BeanLifeCycleAllInOne.class.getSimpleName().equalsIgnoreCase(beanName)) {
            log.info("AfterBeanInitial.postProcessBeforeInitialization -- {}", beanName);
        }
        return bean;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        if (BeanLifeCycleAllInOne.class.getSimpleName().equalsIgnoreCase(beanName)) {
            log.info("AfterBeanInitial.postProcessAfterInitialization -- {}", beanName);
        }
        return bean;
    }
}

```

## 测试输出

```log
2019-09-01 11:05:36.426  INFO 53094 --- [           main] m.r.b.l.bean.BeanLifeCycleAllInOne       : BeanNameAware.setBeanName -- beanLifeCycleAllInOne
2019-09-01 11:05:36.426  INFO 53094 --- [           main] m.r.b.l.bean.BeanLifeCycleAllInOne       : BeanFactoryAware.setBeanFactory
2019-09-01 11:05:36.426  INFO 53094 --- [           main] m.r.b.l.bean.BeanLifeCycleAllInOne       : EnvironmentAware.setEnvironment
2019-09-01 11:05:36.426  INFO 53094 --- [           main] m.r.b.l.bean.BeanLifeCycleAllInOne       : ResourceLoaderAware.setResourceLoader
2019-09-01 11:05:36.426  INFO 53094 --- [           main] m.r.b.l.bean.BeanLifeCycleAllInOne       : ApplicationContextAware.setApplicationContext
2019-09-01 11:05:36.426  INFO 53094 --- [           main] m.r.b.lifecycle.bean.AfterBeanInitial    : AfterBeanInitial.postProcessBeforeInitialization -- beanLifeCycleAllInOne
2019-09-01 11:05:36.426  INFO 53094 --- [           main] m.r.b.l.bean.BeanLifeCycleAllInOne       : BeanLifeCycleAllInOne.postConstruct
2019-09-01 11:05:36.427  INFO 53094 --- [           main] m.r.b.lifecycle.bean.AfterBeanInitial    : AfterBeanInitial.postProcessAfterInitialization -- beanLifeCycleAllInOne

...

2019-09-01 11:06:57.845  INFO 53094 --- [      Thread-16] m.r.b.l.bean.BeanLifeCycleAllInOne       : BeanLifeCycleAllInOne.preDestroy
```

## 参考

1. [ Java 程序员必备的一些流程图 -  芋道源码](https://mp.weixin.qq.com/s/8D4OXsU7CoSHF68LtaF2YA)
1. [Interface Aware](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/beans/factory/Aware.html)
