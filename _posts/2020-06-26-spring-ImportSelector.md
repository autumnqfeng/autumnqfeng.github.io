---
layout:     post
title: "springboot之ImportSelector"
subtitle: "如何实现动态批量装载Bean？"
date:       2020-06-26 21:58:00
author:     "Autu"
header-img: "img/post-bg-2015.jpg"
header-mask: 0.3
catalog:    true
tags:
  - 分布式
  - springboot
  - spring
---

## Spring Boot之ImportSelector

### ImportSelector介绍

ImportSelector这个接口不是有了springboot之后才有的，它是在org.springframework.context.annotation这个包下，随着spring-context包3.1版本发布的时候出现的。

### 用法和用途

其用途比较简单，可以根据启动的相关环境配置来决定让哪些类能够被Spring容器初始化。

#### 举个例子:

一个是事务配置相关的TransactionManagementConfigurationSelector，其实现如下:

```java
@Override
protected String[] selectImports(AdviceMode adviceMode) {
   switch (adviceMode) {
      case PROXY:
         return new String[] {AutoProxyRegistrar.class.getName(),
               ProxyTransactionManagementConfiguration.class.getName()};
      case ASPECTJ:
         return new String[] {determineTransactionAspectClass()};
      default:
         return null;
   }
}
```

总结下来的用法应该是这样：

1. 定义一个Annotation, Annotation中定义一些属性，到时候会根据这些属性的不同返回不同的class数组。
2. 在selectImports方法中，获取对应的Annotation的配置，根据不同的配置来初始化不同的class。
3. 实现ImportSelector接口的对象应该是在Annotation中由@Import Annotation来引入。这也就意味着，一旦启动了注解，那么就会实例化这个对象。

### 实践

比如我有一个需求，要根据当前环境是测试还是生产来决定我们的日志是输出到本地还是阿里云。

假设我定义两个logger ：<code>LoggerA</code> 和 <code>LoggerB</code> ,两个logger实现 Logger接口。我们再定义一个Annotation。

#### Annotation（<code>@EnableMyLogger</code>）的定义：

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Import({LoggerSelector.class})
public @interface EnableMyLogger {
}
```

#### <code>LoggerSelector</code>的定义：

```java
public class LoggerSelector implements ImportSelector {
    @Override
    public String[] selectImports(AnnotationMetadata importingClassMetadata) {
        List<String> classNames = new ArrayList<>();
        if (testEnv()) {
            classNames.add(LoggerA.class.getName());
        } else {
            classNames.add(LoggerB.class.getName());
        }
        return classNames.toArray(new String[0]);
    }

    private boolean testEnv() {
        return false;
    }
}
```

#### 配置类定义：

```java
@Configuration
@EnableMyLogger
public class SelectorConfig {
}
```

#### 测试类定义：

```java
public class MainTest {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(SelectorConfig.class);
        System.out.println(applicationContext.getBean(Logger.class));
    }
}
```

测试结果：

![1593232560898](<http://123.57.45.66/images/autu_blog/micro_service/springboot/2020-06-26-spring-ImportSelector/1593232560898.png>)

事实证明，<code>testEnv()</code>方法返回值为false，确实<code>LoggerA</code>没有注入到容器，<code>LoggerB</code>注入到容器。