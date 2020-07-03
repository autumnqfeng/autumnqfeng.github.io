---
layout:     post
title: "教你手写一个spring-boot-starter组件"
subtitle: "spring-boot核心之Starter组件"
date:       2020-07-03 23:20:00
author:     "Autu"
header-img: "img/post-bg-web.jpg"
header-mask: 0.3
catalog:    true
tags:
  - 分布式
  - 微服务
  - springboot
---

## 【springboot】手写spring-boot-starter组件

### 前言

starter会把所有用到的依赖包都包含进来，避免开发者自己去引入依赖所带来的麻烦。

虽然不同的starter实现起来各有差异，但是他们基本上都会使用到两个相同的内容：<code>ConfigurationProperties</code>和<code>AutoConfiguration</code>。

![1593781249493](http://123.57.45.66/images/autu_blog/micro_service/springboot/2020-07-03-write-spring-boot-starter/1593781249493.png)

Starter是一组可以让你很方便的在应用增加的依赖关系描述符的集合。或者可以这样理解，平时我们开发的时候很多情况下都会有一个模块依赖另一个模块，这个时候我们一般都是采用maven的模块依赖，进行模块的依赖，但是这种情况下我们完全可以采用Starter的方式，将需要被依赖的模块用Starter的方式去开发，最后直接引入一个Starter也可以达到这样的效果。

### 命名规则

> 由于SpringBoot官方本身就提供了很多Starter，为了区别那些是官方的，哪些是第三方的，所以SpringBoot官方提出：
>
> 第三方提供的Starter统一用<font color='red'>xxx-spring-boot-starter</font>
>
> 而官方提供的Starter统一用<font color='red'>spring-boot-starter-xxx</font>。

### 需求

下边我们将以Redisson为例，实现一个简易版的starter组件，通过starter组件将<code>RedissonClient</code>所需的jar和bean依赖到我们当前项目。

### 项目结构

![1593784887682](http://123.57.45.66/images/autu_blog/micro_service/springboot/2020-07-03-write-spring-boot-starter/1593788881948.png)

### 创建Starter

首先我们创建一个<code>redisson-spring-boot-starter</code>的项目，并添加<code>redisson</code>依赖和<code>spring-boot-starter</code>依赖，如下

```xml
<?xml version="1.0" encoding="UTF-8"?>

<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>com.autu.example.redisson</groupId>
  <artifactId>redisson-spring-boot-starter</artifactId>
  <version>1.0-SNAPSHOT</version>

  <name>redisson-spring-boot-starter</name>
  <!-- FIXME change it to the project's website -->
  <url>http://www.example.com</url>

  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter</artifactId>
      <version>2.3.1.RELEASE</version>
      <!-- 禁止传递依赖 -->
      <optional>true</optional>
    </dependency>
    <dependency>
      <groupId>org.redisson</groupId>
      <artifactId>redisson</artifactId>
      <version>3.13.1</version>
    </dependency>
      <!-- 配置文件提示，需加此依赖 -->
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-configuration-processor</artifactId>
      <version>2.3.1.RELEASE</version>
    </dependency>
  </dependencies>
</project>
```

### 编写配置类

创建一个ConfigurationProperties用于保存配置信息

```java
@ConfigurationProperties(prefix = "auto.redisson")
public class RedissonProperties {

    /** Redis server host */
    private String host = "localhost";

    /** Redis server port */
    private int port = 6379;

    /** 连接超时时间 */
    private int timeout;

    /** 是否启用ssl支持 */
    private boolean ssl;
    
    ...
}
```

### 创建自动化配置类

- 创建一个<code>AutoConfiguration</code>，引用定义好的配置信息
- 在<code>AutoConfiguration</code>中实现bean的注入以及配置信息的读取
- 把这个类加入spring.factories配置文件中进行声明

```java
@ConditionalOnClass(Redisson.class)
@EnableConfigurationProperties(RedissonProperties.class)
@Configuration
public class RedissonAutoConfiguration {

    @Bean
    RedissonClient redissonClient(RedissonProperties redissonProperties) {
        Config config = new Config();
		// 判断是否启用ssl
        String prefix = redissonProperties.isSsl() ? "rediss://" : "redis://";

        String host = redissonProperties.getHost();
        int port = redissonProperties.getPort();

        config.useSingleServer()
                .setAddress(prefix + host + ":" + port)
                .setConnectTimeout(1000 * 30);

        return Redisson.create(config);
    }

}
```

### 创建<code>spring.factories</code> 

```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  com.autu.example.redisson.RedissonAutoConfiguration
```

### 创建<code>additional-spring-configuration-metadata.json</code> 

<code>additional-spring-configuration-metadata.json</code>的作用：描述配置信息，在其他项目依赖当前starter组件时起到提示的作用。

```xml
<!-- 配置文件提示需在starter中加入该依赖 -->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-configuration-processor</artifactId>
  <version>2.3.1.RELEASE</version>
</dependency>
```

```json
{
  "properties": [
    {
      "name": "autu.redissin.host",
      "type": "java.lang.String",
      "description": "redis服务器地址.",
      "defaultValue": "localhost"
    },{
      "name": "autu.redisson.port",
      "type": "java.lang.Integer",
      "description": "redis服务器端口.",
      "defaultValue": 6379
    }
  ]
}
```

![1593785391719](http://123.57.45.66/images/autu_blog/micro_service/springboot/2020-07-03-write-spring-boot-starter/1593788828204.png)

接下来，我们通过运行<code>mvn install</code>命令，将这个项目打成jar包<font color='red'>部署到本地仓库</font>，提供给另一个服务调用。

### 测试

创建一个测试项目，依赖<code>redisson-spring-boot-starter</code> 。

#### <code>application.properties</code>配置文件

```properties
auto.redisson.host=127.0.0.1
auto.redisson.port=6379
auto.redisson.timeout=10000
auto.redisson.ssl=false
```

#### 测试类

```java
@RestController
public class HelloController {

    @Autowired
    RedissonClient redissonClient;

    @GetMapping("/test")
    public String say() {
        RBucket<Object> bucket = redissonClient.getBucket("name");
        if (bucket.get() == null) {
            bucket.set("bucket");
        }
        return bucket.get().toString();
    }
}
```

#### 测试结果

![1593789302601](http://123.57.45.66/images/autu_blog/micro_service/springboot/2020-07-03-write-spring-boot-starter/1593789302601.png)

从结果中，我们看到starter中定义的<code>RedissonClient</code>已成功注入到测试项目中。