---
layout:     post
title: "springboot之Enable模块驱动"
subtitle: "如何实现动态装配一组Bean？"
date:       2020-06-26 19:58:00
author:     "Autu"
header-img: "img/home-bg-o.jpg"
header-mask: 0.3
catalog:    true
tags:
  - 分布式
  - springboot
  - spring
---

## Spring Boot 之 Enable模块驱动

### 前言

> <code>@Enable*</code>是 Spring 3.X 产生的注解，Spring Framework 3.0是一个里程碑的时代，其中之一就是取代xml配置方式。

<code>@Enable*</code>的作用：自动完成相关组件的bean的装配

![1593163602430](http://123.57.45.66/images/autu_blog/micro_service/springboot/2020-06-26-spring-@Enable-@Import/1593163602430.png)

### <code>@EnableScheduling</code>实验

下面我们以<code>@EnableScheduling</code>为例，研究一下，Enable模块驱动到底如何将所需的bean注入到容器中的。

首先，我们创建一个<code>TaskConfig</code>类，并在配置类上加<code>@EnableScheduling</code>注解：

```java
@ComponentScan("com.example.springboot.demo")
@EnableScheduling
@Configuration
public class TaskConfiguration {
}
```

然后创建<code>TaskService</code>类，测试定时任务

```java
@Service
public class TaskService {
    @Scheduled(fixedRate = 3000)
    public void reportCurrentTime(){
        System.out.println("current Time:"+new Date());
    }
}
```

最后创建启动类

```java
public class TaskMain {
    public static void main(String[] args) {
        ApplicationContext applicationContext=new AnnotationConfigApplicationContext(TaskConfiguration.class);
    }
}
```

以上3个类均在同一个包下。

下面我们看一下执行结果：

![1593164854214](http://123.57.45.66/images/autu_blog/micro_service/springboot/2020-06-26-spring-@Enable-@Import/1593164854214.png)

### <code>@EnableScheduling</code>原理

首先，我们到<code>@EnableScheduling</code>里面看下，注解是如何定义的

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Import(SchedulingConfiguration.class)
@Documented
public @interface EnableScheduling {

}
```

在<code>@EnableScheduling</code>中，通过<code>@Import</code>注解导入定时任务的配置类<code>SchedulingConfiguration</code>。

```java
@Configuration
@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
public class SchedulingConfiguration {

   @Bean(name = TaskManagementConfigUtils.SCHEDULED_ANNOTATION_PROCESSOR_BEAN_NAME)
   @Role(BeanDefinition.ROLE_INFRASTRUCTURE)
   public ScheduledAnnotationBeanPostProcessor scheduledAnnotationProcessor() {
      return new ScheduledAnnotationBeanPostProcessor();
   }

}
```

在<code>SchedulingConfiguration</code>类中，通过<code>@Bean</code>将定时任务注解处理的实例注入到容器中，因此<code>TaskService</code>中<code>@Scheduled(fixedRate = 3000)</code>会定时3秒执行一次。

### <code>@Import</code>详解

> 在应用中，有时没有把某个类注入到IOC容器中，但在运用的时候需要获取该类对应的bean，此时就需要用到@Import注解。

#### 直接导入类

首先，创建两个类，不用注解注入到IOC容器中，在应用的时候再导入到当前容器中。

<code>Dog</code>类：

```java
public class Dog {
}
```

<code>Cat</code>类：

```java
public class Cat {
}
```

然后，创建**配置类**，在配置类中需要获取<code>Dog</code>和<code>Cat</code>类，需要用到<code>@Import</code>注解将这2个类注入到当前容器中。

```java
@Import({Dog.class, Cat.class})
@Configuration
public class ImportConfig {
}
```

创建启动类，测试<code>Cat</code>和<code>Dog</code>是否注入到了容器中

```java
public class MainTest {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(ImportConfig.class);
        System.out.println(applicationContext.getBean(Dog.class));
        System.out.println(applicationContext.getBean(Cat.class));
    }
}
```

执行结果：

![1593173239572](http://123.57.45.66/images/autu_blog/micro_service/springboot/2020-06-26-spring-@Enable-@Import/1593173239572.png)

事实证明，已经注入到了容器中。

#### 导入配置类

现在在一个配置类中进行配置bean，然后在需要的时候，只需要导入这个配置就可以了，最后输出结果相同。

<code>MyConfig</code> 配置类：

```java
public class MyConfig {

    @Bean
    public Dog dog() {
        return new Dog();
    }

    @Bean
    public Cat cat() {
        return new Cat();
    }
}
```

当前配置类：

```java
// MyConfig、Dog、Cat都将注入到容器中
@Import(MyConfig.class)
@Configuration
public class ImportConfig {
}
```

启动类：

```java
public class MainTest {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(ImportConfig.class);
        System.out.println(applicationContext.getBean(Dog.class));
        System.out.println(applicationContext.getBean(Cat.class));
    }
}
```

执行结果：

![1593174553603](http://123.57.45.66/images/autu_blog/micro_service/springboot/2020-06-26-spring-@Enable-@Import/1593174553603.png)

### 自定义<code>@EnableMyConfig</code>

通过分析<code>@EnableScheduling</code>和<code>@Import</code>，自定义<code>@EnableMyConfig</code>加强理解。

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Import(MyConfig.class)
public @interface EnableMyConfig {
}
```

在当前配置类中加上<code>@EnableMyConfig</code>注解，测试结果同<code>@Import</code>作用在当前配置类结果一样。

```java
@EnableMyConfig
@Configuration
public class ImportConfig {
}
```

启动类：

```java
public class MainTest {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(ImportConfig.class);
        System.out.println(applicationContext.getBean(Dog.class));
        System.out.println(applicationContext.getBean(Cat.class));
    }
}
```

执行结果：

![1593174350786](http://123.57.45.66/images/autu_blog/micro_service/springboot/2020-06-26-spring-@Enable-@Import/1593174350786.png)

### 总结

- 在当前项目配置类中通过<code>@Enable*</code>注解引入需要的模块
- Enable中都会将各模块中的配置类通过<code>@Import</code>导入到当前项目配置类