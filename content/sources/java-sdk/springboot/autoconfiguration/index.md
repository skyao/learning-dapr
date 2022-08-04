---
title: "spring auto configuration"
linkTitle: "auto configuration"
weight: 10
date: 2022-07-24
description: >
  sdk子项目：springboot集成
---

### meta-inf

按照 springboot 的标准做法，`src/main/resources/META-INF/spring.factories` 文件内容如下:

```java
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
io.dapr.springboot.DaprAutoConfiguration
```

### DaprAutoConfiguration

DaprAutoConfiguration 的内容非常简单：

```java
@Configuration
@ConditionalOnWebApplication
@ComponentScan("io.dapr.springboot")
public class DaprAutoConfiguration {
}
```

### DaprBeanPostProcessor

DaprBeanPostProcessor 用来处理 dapr 注解。

```java
@Component
public class DaprBeanPostProcessor implements BeanPostProcessor {

  private static final ObjectMapper MAPPER = new ObjectMapper();

  private final EmbeddedValueResolver embeddedValueResolver;

  DaprBeanPostProcessor(ConfigurableBeanFactory beanFactory) {
    embeddedValueResolver = new EmbeddedValueResolver(beanFactory);
  }
  ......
}
```

BeanPostProcessor 接口的 postProcessBeforeInitialization() 的说明如下：

> 在任何 Bean 初始化回调（如 InitializingBean 的 `afterPropertiesSet` 或自定义 `init-method` ）之前，将此 BeanPostProcessor 应用于给定的新 Bean 实例。
> 该 bean 将已经被填充了属性值。返回的 Bean 实例可能是一个围绕原始 Bean 的包装器。

也就是每个 bean 在初始化后都会调用这个方法以便植入我们需要的逻辑，如在这里就需要扫描 bean 是否带有 dapr 的 topic 注解： 

```java
  @Override
  public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
    if (bean == null) {
      return null;
    }

    subscribeToTopics(bean.getClass(), embeddedValueResolver);

    return bean;
  }
```

subscribeToTopics() 方法的具体实现后面再详细看，期间还有规则匹配的实现代码。

postProcessAfterInitialization() 方法没有特殊逻辑，简单返回原始bean：

```java
  @Override
  public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
    return bean;
  }
```