---
layout:     post
title: "Spring Boot Actuator:健康检查、审计、统计和监控"
subtitle: "Spring Boot Actuator"
date:       2020-07-04 23:20:00
author:     "Autu"
header-img: "img/post-bg-web.jpg"
header-mask: 0.3
catalog:    true
tags:
  - 分布式
  - 微服务
  - springboot
---

## Spring Boot Actuator:健康检查、审计、统计和监控

Spring Boot Actuator可以帮助你监控和管理Spring Boot应用，比如健康检查、审计、统计和HTTP追踪等。所有这些特性可以通过JMX或者HTTP endpoint来获得。

Actuator同时还可以与外部应用监控系统整合，比如[Prometheus](https://prometheus.io/), [Graphite](https://graphiteapp.org/), [DataDog](https://www.datadoghq.com/), [Influx](https://www.influxdata.com/), [Wavefront](https://www.wavefront.com/), [New Relic](https://newrelic.com/)等。这些系统提供了非常好的仪表盘、图标、分析和告警等功能，使得你可以通过统一的接口轻松的监控和管理你的应用。

Actuator使用[Micrometer](http://micrometer.io/)来整合上面提到的外部应用监控系统。这使得只要通过非常小的配置就可以集成任何应用监控系统。

我将把Spring Boot Actuator教程分为两部分：

- 第一部分（本文）教你如何配置Actuator和通过Http endpoints来进入这些特征。
- 第二部分教你如何整合Actuator和外部应用监控系统。

### 创建一个有Actuator的Spring Boot工程

首先让我们建一个依赖Actuator的简单应用。maven依赖如下：

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
</dependencies>
```

### 使用Actuator Endpoints来监控应用

Actuator创建了所谓的**endpoint**来暴露HTTP或者JMX来监控和管理应用。

**举个例子**

- <code>/health</code>的endpoint，提供了关于应用健康的基础信息。
- <code>/metrics</code>endpoint展示了几个有用的度量信息，比如JVM内存使用情况、系统CPU使用情况、打开的文件等。
- <code>/logger</code>endpoint展示了应用的日志和可以让你在运行时改变日志等级。

**值得注意的是，每一个actuator endpoint都可以显示的打开或关闭。此外这些endpoint也需要通过HTTP或者JMX暴露出来，使得它们能被远程进入。**

让我们运行应用并且尝试进入默认通过HTTP暴露的打开状态的actuator endpoints。之后我们将学习如何打开更多的endpoints并且通过HTTP暴露它们。



### 创建应用

让我们启动actuator应用，应用默认使用<code>8080</code>端口运行。

![1593846246837](http://123.57.45.66/images/autu_blog/micro_service/springboot/2020-07-04-spring-boot-actuator/1593846246837.png)

启动成功之后，可以通过<http://localhost:8080/actuator>来展示所有通过HTTP暴露的endpoints。

![1593847652908](http://123.57.45.66/images/autu_blog/micro_service/springboot/2020-07-04-spring-boot-actuator/1593847652908.png)

打开<http://localhost:8080/actuator/health>，则会显示如下内容:

![1593847708251](http://123.57.45.66/images/autu_blog/micro_service/springboot/2020-07-04-spring-boot-actuator/1593847708251.png)

状态将是<code>UP</code>只要应用是健康的，如果应用不健康将会显示<code>DOWN</code>，比如与仪表盘的连接异常或者缺失磁盘空间等。下一节我们将学习Spring Boot如何决定应用的健康和如何修复这些健康问题。

<code>info</code> endpoint（<http://localhost:8080/actuator/info>）展示了关于应用的一般信息，这些信息从编译文件，比如<code>META-INFO/build-info.properties</code>，或者GIT文件，比如<code>git.properties</code>或者任何环境的property中获取。你将在下一节中学习如何改变这个endpoint的输出。

**默认，只有<code>health</code>和<code>info</code>通过HTTP暴露了出来。**这也是为什么<code>/actuator</code>页面只展示了<code>health</code>和<code>info</code>endpoints。我们将学习如何暴露其他endpoint。首先，让我们看看其他的endpoints是什么。



### 常用actuator endpoints列表

以下是一些非常有用的actuator endpoints列表。你可以在[official documentation](https://docs.spring.io/spring-boot/docs/current/reference/html/production-ready-endpoints.html)上面看到完整的列表。

| **Endpoint ID** | **Description**                                              |
| --------------- | ------------------------------------------------------------ |
| auditevents     | 显示应用暴露的审计事件 (比如认证进入、订单失败)              |
| info            | 显示应用的基本信息                                           |
| health          | 显示应用的健康状态                                           |
| metrics         | 显示应用多样的度量信息                                       |
| loggers         | 显示和修改配置的loggers                                      |
| logfile         | 返回log file中的内容(如果logging.file或者logging.path被设置) |
| httptrace       | 显示HTTP足迹，最近100个HTTP request/repsponse                |
| env             | 显示当前的环境特性                                           |
| flyway          | 显示数据库迁移路径的详细信息                                 |
| liquidbase      | 显示Liquibase 数据库迁移的详细信息                           |
| shutdown        | 让你逐步关闭应用                                             |
| mappings        | 显示所有的@RequestMapping路径                                |
| scheduledtasks  | 显示应用中的调度任务                                         |
| threaddump      | 执行一个线程dump                                             |
| heapdump        | 返回一个GZip压缩的JVM堆dump                                  |



### 打开和关闭Actuator Endpoints

默认，上述所有的endpoints都是打开的，除了<code>shutdown</code>endpoint。

你可以通过设置<code>management.endpoint.<id>.enabled to true or false</code>（<code>id</code>是endpoint的id）来决定打开还是关闭一个actuator endpoint。

举个例子，要想打开<code>shudown</code>endpoint，增加以下内容在你的<code>application.properties</code>文件中：

```properties
management.endpoint.shutdown.enabled=true
```



### 暴露Actuator Endpoints

默认，所有的actuator endpoint通过JMX被暴露，而通过HTTP暴露的只有<code>health</code>和<code>info</code>。

- 通过HTTP暴露actuator endpoints。

  ```properties
  # Use "*" to expose all endpoints, or a comma-separated list to expose selected ones
  management.endpoints.web.exposure.include=health,info 
  management.endpoints.web.exposure.exclude=
  ```

- 通过JMX暴露actuator endpoints。

  ```properties
  # Use "*" to expose all endpoints, or a comma-separated list to expose selected ones
  management.endpoints.jmx.exposure.include=*
  management.endpoints.jmx.exposure.exclude=
  ```

通过设置<code>management.endpoints.web.exposure.include</code>为<code>*</code>，我们可以在<http://localhost:8080/actuator>页面看到如下内容。

```json
{
    "_links":{
        "self":{
            "href":"http://localhost:8080/actuator",
            "templated":false
        },
        "beans":{
            "href":"http://localhost:8080/actuator/beans",
            "templated":false
        },
        "caches-cache":{
            "href":"http://localhost:8080/actuator/caches/{cache}",
            "templated":true
        },
        "caches":{
            "href":"http://localhost:8080/actuator/caches",
            "templated":false
        },
        "health":{
            "href":"http://localhost:8080/actuator/health",
            "templated":false
        },
        "health-path":{
            "href":"http://localhost:8080/actuator/health/{*path}",
            "templated":true
        },
        "info":{
            "href":"http://localhost:8080/actuator/info",
            "templated":false
        },
        "conditions":{
            "href":"http://localhost:8080/actuator/conditions",
            "templated":false
        },
        "shutdown":{
            "href":"http://localhost:8080/actuator/shutdown",
            "templated":false
        },
        "configprops":{
            "href":"http://localhost:8080/actuator/configprops",
            "templated":false
        },
        "env":{
            "href":"http://localhost:8080/actuator/env",
            "templated":false
        },
        "env-toMatch":{
            "href":"http://localhost:8080/actuator/env/{toMatch}",
            "templated":true
        },
        "loggers":{
            "href":"http://localhost:8080/actuator/loggers",
            "templated":false
        },
        "loggers-name":{
            "href":"http://localhost:8080/actuator/loggers/{name}",
            "templated":true
        },
        "heapdump":{
            "href":"http://localhost:8080/actuator/heapdump",
            "templated":false
        },
        "threaddump":{
            "href":"http://localhost:8080/actuator/threaddump",
            "templated":false
        },
        "metrics-requiredMetricName":{
            "href":"http://localhost:8080/actuator/metrics/{requiredMetricName}",
            "templated":true
        },
        "metrics":{
            "href":"http://localhost:8080/actuator/metrics",
            "templated":false
        },
        "scheduledtasks":{
            "href":"http://localhost:8080/actuator/scheduledtasks",
            "templated":false
        },
        "mappings":{
            "href":"http://localhost:8080/actuator/mappings",
            "templated":false
        }
    }
}
```



### 解析常用的actuator endpoint

#### <code>/health</code> endpoint

<code>/health</code> endpoint通过合并几个健康指数检查应用的健康情况。

Spring Boot Actuator有几个预定义的健康指标，比如<code>DataSourceHealthIndicator</code>、<code>DiskSpaceHealthIndicator</code>、<code>MongoHealthIndicator</code>、<code>RedisHealthIndicator</code>、<code>CassandraHealthInditor</code>等。它使用这些健康指标作为健康检查的一部分。

举个例子，如果你的应用使用<code>Redis</code>，<code>RedisHealthIndicator</code>将被当作检查的一部分。如果使用<code>Mongo</code>，那么<code>MongoHealthIndicator</code>将被当作检查的一部分。

你也可以关闭特定的健康检查指标，比如在properties中使用如下命令：

```properties
management.health.mongo.enabled=false
```

默认，所有的这些健康指标被当做检查的一部分。



##### 显示详细的健康信息

<code>health</code> endpoint只展示了简单的<code>UP</code>和<code>DOWN</code>状态。

为了获得健康检查中所有指标的详细信息，你可以通过在<code>application.properties</code>中增加如下内容：

```properties
management.endpoint.health.show-details=always
```

一旦打开上述开关，你可以在<http://localhost:8080/actuator/health>中看到如下详细内容：

![1593852424583](http://123.57.45.66/images/autu_blog/micro_service/springboot/2020-07-04-spring-boot-actuator/1593852424583.png)

<code>health</code> endpoint现在包含了<code>DiskSpaceHealthIndicator</code>。

如果你的应用包含Redis，<code>health</code> endpoints将显示如下内容：

![1593853662992](http://123.57.45.66/images/autu_blog/micro_service/springboot/2020-07-04-spring-boot-actuator/1593853662992.png)



##### 创建一个自定义的健康指标

你可以通过实现<code>HealthIndicator</code>接口来定义一个健康指标，或者继承<code>AbstractHealthIndicatior</code>类。

```java
package com.example.actuator.springbootactuator;

import org.springframework.boot.actuate.health.AbstractHealthIndicator;
import org.springframework.boot.actuate.health.Health;
import org.springframework.stereotype.Component;

@Component
public class CustomHealthIndicator extends AbstractHealthIndicator {
    @Override
    protected void doHealthCheck(Health.Builder builder) throws Exception {
        // Use the builder to build the health status details that should be reported.
        // If you throw an exception, the status will be DOWN with the exception message.

        builder.up()
                .withDetail("app", "Alive and Kicking")
                .withDetail("error", "Nothing! I'm good.");
    }
}
```

一旦你增加上面的健康指标到你的应用中去，<code>health</code> endpoints将展示如下细节：

![1593854466984](http://123.57.45.66/images/autu_blog/micro_service/springboot/2020-07-04-spring-boot-actuator/1593854466984.png)

#### <code>/metrics</code> endpoint

<code>metrics</code> endpoint展示了你可以追踪的所有度量。

![1593862185463](http://123.57.45.66/images/autu_blog/micro_service/springboot/2020-07-04-spring-boot-actuator/1593862228187.png)

想获得每个度量的详细信息，你需要传递度量的名称到URL中，像[http://localhost:8080/actuator/metrics/{MetricName}](http://localhost:8080/actuator/metrics/%7BMetricName)

举个例子，获得<code>system.cpu.usage</code>的详细信息，使用URL <http://localhost:8080/actuator/metrics/system.cpu.usage>。

它将显示如下内容：

![1593862585251](http://123.57.45.66/images/autu_blog/micro_service/springboot/2020-07-04-spring-boot-actuator/1593862585251.png)

#### <code>/loggers</code> endpoint

<code>loggers</code> endpoint，可以通过访问<http://localhost:8080/actuator/loggers>来进入。他展示了应用中可配置的loggers的列表和相关日志等级。

你同样能够使用[http://localhost:8080/actuator/loggers/{name}](http://localhost:8080/actuator/loggers/%7Bname)来展示特定logger的细节。

举个例子，为了获得<code>root</code> logger的细节，你可以使用<http://localhost:8080/actuator/loggers/ROOT>：

![1593863101755](http://123.57.45.66/images/autu_blog/micro_service/springboot/2020-07-04-spring-boot-actuator/1593863101755.png)

##### 在运行时改变日志等级

<code>loggers</code> endpoint也允许你在运行时改变应用的日志等级。

举个例子，为了改变<code>root</code> logger的等级为<code>DEBUG</code>，发送一个<code>POST</code>请求到<http://localhost:8080/actuator/loggers/ROOT>，加入如下参数

```json
{
   "configuredLevel": "DEBUG"
}
```

这个功能对于线上问题的排查非常有用。

同时，你可以通过传递<code>null</code>值给<code>configuredLevel</code>来重置日志等级。



#### <code>/info</code> endpoint

<code>/info</code> endpoint展示了应用的基本信息。它通过<code>META-INF/build-info.properties</code>来获得编译信息，通过<code>git.properties</code>来获得git信息。它同时可以展示任何其他信息，只要这个环境property中含有<code>info</code> key。

你可以增加properties到<code>application.properties</code>中，比如：

```properties
# INFO ENDPOINT CONFIGURATION
info.app.name=@project.name@
info.app.description=@project.description@
info.app.version=@project.version@
info.app.encoding=@project.build.sourceEncoding@
info.app.java.version=@java.version@
```

注意，我使用了Spring Boot的[Automatic property expansion](https://docs.spring.io/spring-boot/docs/current/reference/html/howto-properties-and-configuration.html#howto-automatic-expansion) 特征来扩展来自maven工程的properties。

一旦你增加上面的properties，<code>info</code> endpoint将展示如下信息：

![1593873141517](http://123.57.45.66/images/autu_blog/micro_service/springboot/2020-07-04-spring-boot-actuator/1593873141517.png)



### 注解方式自定义Endpoint

#### 编写自定义endpoint

```java
@Endpoint(id = "my-endpoint")
public class MyEndpoint {

    @ReadOperation
    public Map<String, Object> endpoint() {
        Map<String, Object> map = new HashMap<>(16);
        map.put("当前时间：", new Date().toString());
        map.put("message", "this is my endpoint");
        return map;
    }

}
```

#### 编写配置类

```java
@Configuration
public class EndpointConfiguration {

    @Bean
    public MyEndpoint endpoint() {
        return new MyEndpoint();
    }
}
```

#### 结果

![1593907563689](http://123.57.45.66/images/autu_blog/micro_service/springboot/2020-07-04-spring-boot-actuator/1593907563689.png)

#### 注意

- <code>@EndPoint</code>中的id不能使用驼峰法，需要以-分割
- Spring Boot会去扫描<code>@EndPoint</code>注解下的<code>@ReadOperation</code>，<code>@WriteOperation</code>，<code>@DeleteOperation</code>注解，分别对应生成<code>Get/Post/Delete</code>的Mapping。注解中有个`produces`参数，可以指定media type, 如：<code>application/json</code>等。



### 使用Spring Security来保证Actuator Endpoints安全

Actuator endpoints是敏感的，必须保障进入是被授权的。如果Spring Security是包含在你的应用中，那么endpoint是通过HTTP认证被保护起来的。

如果没有，你可以增加以下依赖到你的应用中去：

```xml
<dependency>
   <groupId>org.springframework.boot</groupId>
   <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

接下来让我们看一下如何覆写spring security配置，并且定义你自己的进入规则。

下面的例子展示了一个简单的spring security配置。它使用叫做<code>EndPointRequest</code>的<code>RequestMatcher</code>工厂模式来配置Actuator endpoints进入规则。

```java
package com.example.actuator.springbootactuator;

import org.springframework.boot.actuate.autoconfigure.security.servlet.EndpointRequest;
import org.springframework.boot.actuate.context.ShutdownEndpoint;
import org.springframework.boot.autoconfigure.security.servlet.PathRequest;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;

@Configuration
public class ActuatorSecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
                .authorizeRequests()
                .requestMatchers(EndpointRequest.to(ShutdownEndpoint.class))
                .hasRole("ACTUATOR_ADMIN")
                .requestMatchers(EndpointRequest.toAnyEndpoint())
                .permitAll()
                .requestMatchers(PathRequest.toStaticResources().atCommonLocations())
                .permitAll()
                .antMatchers("/")
                .permitAll()
                .antMatchers("/**")
                .authenticated()
                .and().httpBasic();
    }

}
```

为了能够测试以上的配置，你可以在`application.propert`中增加spring security用户。

```properties
# Spring Security Default user name and password
spring.security.user.name=actuator
spring.security.user.password=actuator
spring.security.user.roles=ACTUATOR_ADMIN
```



下一部分：[Spring Boot Metrics监控之Prometheus&Grafana](<http://autumn200.com/2020/07/05/spring-boot-actuator-metrics/>)

### 翻译源

- [Spring Boot Actuator metrics monitoring with Prometheus and Grafana](https://www.callicoder.com/spring-boot-actuator-metrics-monitoring-dashboard-prometheus-grafana/)