---
layout:     post
title: "微服务网关 - Spring Cloud Gateway"
subtitle: "高性能网关"
date:       2020-07-26 23:13:00
author:     "Autu"
header-img: "img/post-bg-js-module.jpg"
header-mask: 0.3
catalog:    true
tags:
  - 分布式
  - 微服务
  - springcloud
---

## 微服务网关 - Spring Cloud Gateway

在微服务架构中，每个服务都是一个可以独立开发和运行的组件，而一个完整的微服务架构由一系列独立运行的微服务组成。客户端完成一个功能可能需要调用多个服务，因此也会带来一些影响，比如：

* 客户端需要发起多次请求，增加了网络通信的成本及客户端处理的复杂性。
* 服务的鉴权会分布在每个微服务中处理，客户端对于每个服务的调用都需要重复的鉴权。
* 在后端的微服务架构中，可能不同的服务采用的协议不同，比如有HTTP、RPC等。客户端如果需要调用多个服务，需要对不同的协议进行适配。

### API网关的作用

网关可以用来解决这个问题，在客户端与服务端之间增加一个API网关。

网关不仅只是做一个请求转发及服务的整合，有了网关这个统一的入口之后，他还能提供以下功能：

* 针对所有请求进行**统一鉴权**、**限流**、**熔断**、**日志**。
* 协议转化。
* 统一错误码处理。
* 请求转发，并且可以基于网关实现内、外网隔离。

#### 灰度发布

很多公司产品迭代速度非常快，在高频率的迭代模式下，往往会伴随一些风险，比如：

* 新发布的代码出现兼容性问题。
* 新的功能发布后，用户是否能够接受，如果不能，会造成用户流失。
* 代码中存在隐藏的Bug，导致线上故障。

为了规避这些问题，对于有较大的功能改动的版本一般会采取灰度发布的方式来实现平滑过渡。



> 所谓的灰度发布，就是指将要发布的功能先开放给一小部分用户使用，把影响范围控制在一个非常小的范围，比如 A/B Test 就是一种灰度发布的方式，即一部分用户继续使用 A 功能，另外一小部分用户使用新的 B 功能。通过对使用 B 功能的用户进行满意度调查，以及对新发布的代码性能和稳定性指标进行测评，逐步放大该新版本的投放，直到全量或者回滚改版本。



网关是所有请求的入口，因此在网关层可以通过灰度规则进行部分流量的路由，从而实现灰度发布。如下图所示，网关对请求拦截后，会根据分流引擎配置的分流规则进行请求的路由。



![1595748184114](http://123.57.45.66/images/autu_blog/micro_service/springcloud/2020-07-26-spring-cloud-gateway/1595748184114.png)



### Spring Cloud Gateway简介

Spring Cloud Gateway 是 Spring 官方团队研发的 API 网关技术，他的目的是取代 Zuul ，为微服务提供一种简单高效的 API 网关。

Spring Cloud Gateway 是依赖于 Spring Boot 2.0 、Spring WebFlux 和Project Reactor 等技术开发的网关，他不仅提供了统一的路由请求的方式，还基于过滤链的方式提供了网关最基本的功能，例如：安全、监控、埋点和限流等。

#### 优点

- 性能强劲，是Zuul的1.6倍
- 功能强大，内置了很多实用的功能，例如转发、监控、限流等
- 设计优雅，容易扩展

#### 缺点

- 依赖Netty与WebFlux，不是传统的Servlet编程模型，有一定的学习成本
- 不能在Servlet容器下工作，也不能构建成WAR包，即不能将其部署在Tomcat、Jetty等Servlet容器里，只能打成jar包执行
- 不支持Spring Boot 1.x，需2.0及更高的版本



### Spring Cloud Gateway 原理分析

Spring Cloud Gateway 的请求过程如图所示：

![1595756235582](http://123.57.45.66/images/autu_blog/micro_service/springcloud/2020-07-26-spring-cloud-gateway/1595756235582.png)

其中有几个非常重要的概念：

- Route（路由）：它是网关的基本组件，由ID、目标URL、Predicate 集合、Filter 集合组成。
- Predicate（断言）：它是 Java 8 中引入的函数式接口，提供了断言的功能。它可以匹配 HTTP 请求中的任何内容。如果 Predicate 的聚合判断结果为 true，则意味着该请求会被当前 Router 进行转发。
- Filter（过滤器）：为请求提供前置和后置的过滤。

### Spring Cloud Gateway 网关实战

首先准备两个 Spring Boot 应用。

* spring-cloud-gateway-service，模拟一个微服务。
* spring-cloud-gateway-sample，独立的网关服务。

#### spring-cloud-gateway-service

基于 Spring Boot 脚手架构建一个应用，添加 spring-boot-starter-web 依赖。创建一个 `HelloController` 类发布一个接口并启动该应用。

```java
@RestController
public class HelloController {
    
    @GetMapping("/say")
    public String sayHello() {
        return "[spring-cloud-gateway-service]:say Hello";
    }
}
```

#### spring-cloud-gateway-sample

创建 Spring Boot 应用，添加 Spring Cloud Gateway 依赖。

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
```

在 `application.yml` 文件中添加 Gateway 的路由配置。

```yml
spring:
  cloud:
    gateway:
      routes:
        - id: config_route
          predicates:
            - Path=/gateway/** # 路径匹配
          filters:
            - StripPrefix=1 # 跳过前缀
          uri: http://localhost:8080/say # spring-cloud-gateway-service 的访问地址
server:
  port: 8088
```

上述配置中字段的含义说明如下：

* **id**：自定义路由ID，保持唯一。
* **uri**：目标服务器地址，支持普通 URI 及 lb://应用注册服务名称，后者表示从注册中心获取集群服务地址。
* **predicates**：路由条件，根据匹配的结果决定是否执行该请求路由。
* **filters**：过滤规则，包含 `pre` 和 `post` 过滤。其中 `StripPrefix=1` ，表示 Gateway 根据该配置的值去掉 URL 路径中的部分前缀（这里去掉一个前缀，即在转发的目标 URL 中去掉 gateway ）。

启动应用，在控制台可以获得如下信息，可以看到，它并没有依赖 Tomcat ，而是用 `NettyWebServer` 来启动一个服务监听。

```verilog
2020-07-26 16:32:01.375  INFO 13544 --- [           main] o.s.b.web.embedded.netty.NettyWebServer  : Netty started on port(s): 8088
```

![1595752523729](http://123.57.45.66/images/autu_blog/micro_service/springcloud/2020-07-26-spring-cloud-gateway/1595752523729.png)

在浏览器输入 `http://localhost:8088/gateway/say` ，结果：

![1595753192469](http://123.57.45.66/images/autu_blog/micro_service/springcloud/2020-07-26-spring-cloud-gateway/1595753192469.png)

在浏览器输入 `http://localhost:8080/say` ，结果：

![1595753246339](http://123.57.45.66/images/autu_blog/micro_service/springcloud/2020-07-26-spring-cloud-gateway/1595753246339.png)

我们发现，结果是相同的。

### Route Predicate Factories

`Predicate` 是 `Java 8` 提供的一个函数式接口，它允许接受一个参数并返回一个布尔值，可以用于条件过滤、请求参数的校验。

```java
@FunctionalInterface
public interface Predicate<T> {
    boolean test(T t);
}
```

`Spring Cloud Gateway` 默认提供了很多 `Route Predicate Factory`，这些 `Predicate` 会分别匹配 `HTTP` 请求的不同属性，并且多个 `Predicate` 可以通过 `and` 逻辑进行组合。

下面是 `HTTP` 请求的属性对应的 `Predicate` ：

![1595760050850](http://123.57.45.66/images/autu_blog/micro_service/springcloud/2020-07-26-spring-cloud-gateway/1595760050850.png)

#### After Route Predicate

在指定时间之后的请求会匹配该路由。

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: after_route
          uri: http://localhost:8080/say
          predicates:
            - After=2020-07-26T16:30:00+08:00[Asia/Shanghai]
```

#### Before Route Predicate

在指定时间之前的请求会匹配该路由。

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: before_route
          uri: http://localhost:8080/say
          predicates:
            - Before=2020-07-26T16:30:00+08:00[Asia/Shanghai]
```

#### Between Route Predicate

在指定时间区间内的请求会匹配该路由。

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: between_route
          uri: http://localhost:8080/say
          predicates:
            - Between=2020-07-26T16:30:00+08:00[Asia/Shanghai], 2020-07-27T16:30:00+08:00[Asia/Shanghai]
```

#### Cookie Route Predicate

带有指定Cookie的请求会匹配该路由。

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: cookie_route
          uri: http://localhost:8080/say
          predicates:
            - Cookie=username,macro
```

使用curl工具发送带有cookie为 `username=macro` 的请求可以匹配该路由。

```shell
curl http://localhost:8088/gateway/say --cookie "username=macro"
```

#### Header Route Predicate

带有指定请求头的请求会匹配该路由。

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: header_route
        uri: http://localhost:8080/say
        predicates:
        - Header=X-Request-Id, \d+
```

使用curl工具发送带有请求头为 `X-Request-Id:123` 的请求可以匹配该路由。

```shell
curl http://localhost:8088/gateway/say -H "X-Request-Id:123" 
```

#### Host Route Predicate

带有指定Host的请求会匹配该路由。

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: host_route
          uri: http://localhost:8080/say
          predicates:
            - Host=**.macro.com
```

使用curl工具发送带有请求头为 `Host:www.macro.com` 的请求可以匹配该路由。

```shell
curl http://localhost:8088/gateway/say -H "Host:www.macro.com" 
```

#### Method Route Predicate

发送指定方法的请求会匹配该路由。

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: method_route
        uri: http://localhost:8080/say
        predicates:
        - Method=GET
```

使用curl工具发送 `GET` 请求可以匹配该路由。

```shell
curl http://localhost:8088/gateway/say
```

使用curl工具发送 `POST` 请求无法匹配该路由。

```shell
curl -X POST http://localhost:8088/gateway/say
```

#### Path Route Predicate

发送指定路径的请求会匹配该路由。

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: path_route
          uri: http://localhost:8080/say
          predicates:
            - Path=/gateway/**
```

使用curl工具发送 `/gateway/` 路径请求可以匹配该路由。

```shell
curl http://localhost:8088/gateway/say
```

使用curl工具发送 `/abc/` 路径请求无法匹配该路由。

```shell
curl http://localhost:8088/abc/say
```

#### Query Route Predicate

带指定查询参数的请求可以匹配该路由。

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: query_route
        uri: http://localhost:8080/say/getByUsername
        predicates:
        - Query=username
```

使用curl工具发送带 `username=macro` 查询参数的请求可以匹配该路由。

```shell
curl http://localhost:8088/gateway/say/getByUsername?username=macro
```

使用curl工具发送带不带查询参数的请求无法匹配该路由。

```shell
curl http://localhost:8088/gateway/say/getByUsername
```

#### RemoteAddr Route Predicate

从指定远程地址发起的请求可以匹配该路由。

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: remoteaddr_route
        uri: http://localhost:8080/say/
        predicates:
        - RemoteAddr=192.168.1.1/24
```

使用curl工具从192.168.1.1发起请求可以匹配该路由。

```shell
curl http://localhost:8088/gateway/say/
```

#### Weight Route Predicate

使用权重来路由相应请求，以下表示有80%的请求会被路由到 `localhost:8080`，20%会被路由到 `localhost:8081`。

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: weight_high
        uri: http://localhost:8080
        predicates:
        - Weight=group1, 8
      - id: weight_low
        uri: http://localhost:8081
        predicates:
        - Weight=group1, 2
```

### Gateway Filter Factories

Filter 分为 `Pre` 类型的过滤器和 `Post` 类型的过滤器。

* `Pre` 类型的过滤器在请求转发到后端服务器之前执行，在 `Per` 类型过滤器链中可以做<font color=red>**鉴权**</font>、<font color=red>**限流**</font>等操作。

* `Post` 类型的过滤器在请求完之后、将结果返回客户端之前执行。

在 Spring Cloud Gateway 中内置了很多 Filter，Filter 有两种实现，分别是 `GatewayFilter` 和 `GlobalFilter`。`GlobalFilter` 会应用到所有的路由器上，而 `GatewayFilter` 只会应用到单个路由器或者一个分组的路由器上。

#### AddRequestParameter GatewayFilter

给请求添加参数的过滤器。

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: add_request_parameter_route
          uri: http://localhost:8080
          filters:
            - AddRequestParameter=username, macro
          predicates:
            - Method=GET
```

以上配置会对GET请求添加 `username=macro` 的请求参数，通过curl工具使用以下命令进行测试。

```shell
curl http://localhost:8088/gateway/say/getByUsername
```

相当于发起该请求：

```shell
curl http://localhost:8088/gateway/say/getByUsername?username=macro
```

#### StripPrefix GatewayFilter

对指定数量的路径前缀进行去除的过滤器。

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: strip_prefix_route
        uri: http://localhost:8080
        predicates:
        - Path=/gateway/**
        filters:
        - StripPrefix=1
```

以上配置会把以 `/gateway/` 开头的请求的路径去除一位，通过curl工具使用以下命令进行测试。

```shell
curl http://localhost:8088/gateway/say/
```

相当于发起该请求：

```shell
curl http://localhost:8080/say/
```

#### PrefixPath GatewayFilter

与StripPrefix过滤器恰好相反，会对原有路径进行增加操作的过滤器。

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: prefix_path_route
        uri: http://localhost:8080
        predicates:
        - Method=GET
        filters:
        - PrefixPath=/user
```

以上配置会对所有GET请求添加`/user`路径前缀，通过curl工具使用以下命令进行测试。

```shell
curl http://localhost:8088/gateway/say
```

相当于发起该请求：

```shell
curl http://localhost:8080/user/gateway/say/
```

#### Hystrix GatewayFilter

Hystrix 过滤器允许你将断路器功能添加到网关路由中，使你的服务免受级联故障的影响，并提供服务降级处理。

* 要开启断路器功能，我们需要在 `pom.xml` 中添加Hystrix的相关依赖：

  ```xml
  <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
  </dependency>
  ```

* 然后添加相关服务降级的处理类：

  ```java
  @RestController
  public class FallbackController {
  
      @GetMapping("/fallback")
      public Object fallback() {
          Map<String,Object> result = new HashMap<>();
          result.put("data",null);
          result.put("message","Get request fallback!");
          result.put("code",500);
          return result;
      }
  }
  ```

* 在 `application-filter.yml` 中添加相关配置，当路由出错时会转发到服务降级处理的控制器上：

  ```yaml
  spring:
    cloud:
      gateway:
        routes:
          - id: hystrix_route
            predicates:
              - Method=GET
            filters:
              - name: Hystrix
                args:
                  name: fallbackcmd
                  fallbackUri: forward:/fallback
            uri: http://localhost:8080/say
  ```

* 关闭spring-cloud-gateway-service，调用该地址进行测试：`http://localhost:8088/say`，发现已经返回了服务降级的处理信息。

  ![1595764410272](http://123.57.45.66/images/autu_blog/micro_service/springcloud/2020-07-26-spring-cloud-gateway/1595764410272.png)

#### RequestRateLimiter GatewayFilter

RequestRateLimiter 过滤器可以用于**限流**，使用 `RateLimiter` 实现来确定是否允许当前请求继续进行，如果请求太大默认会返回HTTP 429 -太多请求状态。

* 在 `pom.xml` 中添加相关依赖：

  ```xml
  <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-data-redis-reactive</artifactId>
  </dependency>
  ```

* 添加限流策略，根据访问IP进行限流，定义限流策略类 `IpKeyResolver` ，并注入到spring容器中。

  ```java
  @Component
  public class IpKeyResolver implements KeyResolver {
      @Override
      public Mono<String> resolve(ServerWebExchange exchange) {
          return Mono.just(exchange.getRequest().getRemoteAddress().getHostName());
      }
  }
  ```

* 我们使用Redis来进行限流，所以需要添加Redis和RequestRateLimiter的配置，这里对所有的GET请求都进行了按IP来限流的操作。

  ```yaml
  server:
    port: 8088
  spring:
    redis:
      host: 123.57.45.66
      port: 6379
    cloud:
      gateway:
        routes:
          - id: requestratelimiter_route
            uri: http://localhost:8080/
            filters:
              - name: RequestRateLimiter
                args:
                  redis-rate-limiter.replenishRate: 1 # 每秒允许处理的请求数量
                  redis-rate-limiter.burstCapacity: 2 # 每秒最大处理的请求数量
                  key-resolver: "#{@ipKeyResolver}" # 限流策略，对应策略的Bean
            predicates:
              - Method=GET
  ```

* 多次请求该地址：`http://localhost:8088/say` ，会返回状态码为429的错误；

  ![1595852606905](http://123.57.45.66/images/autu_blog/micro_service/springcloud/2020-07-26-spring-cloud-gateway/1595852606905.png)

#### Retry GatewayFilter

对路由请求进行重试的过滤器，可以根据路由请求返回的HTTP状态码来确定是否进行重试。

* 修改配置文件：

  ```yaml
  server:
    port: 8088
  spring:
    cloud:
      gateway:
        routes:
          - id: retry_route
            uri: http://localhost:8080
            predicates:
              - Method=GET
            filters:
              - name: Retry
                args:
                  retries: 1 # 需要进行重试的次数
                  statuses: BAD_GATEWAY # 返回哪个状态码需要进行重试，返回状态码为5XX进行重试
                  backoff:
                    firstBackoff: 10ms
                    maxBackoff: 50ms
                    factor: 2
                    basedOnPreviousValue: false
  ```

* 修改 `spring-cloud-gateway-service` 中 `HelloController` 的 `/say` 方法，手动抛出异常。

  ```java
  @GetMapping("/say")
  public String sayHello() {
      if (true) {
          throw new RuntimeException("/say, 异常");
      }
      return "[spring-cloud-gateway-service]:say Hello";
  }
  ```

* 当调用返回500时会进行重试，访问测试地址：`http://localhost:8088/say` 

* 可以发现 `spring-cloud-gateway-service` 控制台报错2次，说明进行了一次重试。

  ```verilog
  2020-07-27 21:50:31.910 ERROR 11200 --- [nio-8080-exec-1] o.a.c.c.C.[.[.[/].[dispatcherServlet]    : Servlet.service() for servlet [dispatcherServlet] in context with path [] threw exception [Request processing failed; nested exception is java.lang.RuntimeException: /say, 异常] with root cause
  
  java.lang.RuntimeException: /say, 异常
  	at com.autu.example.springcloudgatewayservice.HelloController.sayHello(HelloController.java:16) ~[classes/:na]
  ```

### 自定义 Predicate Factory

例如有某个服务限制用户只允许在06:00 - 13:00这个时间段内才可以访问，内置的 `Predicate Fatory` 是无法满足这个需求的，所以此时我们就需要自定义能够实现该需求的 `Predicate Fatory` 。

* 创建 `TimeBetweenRoutePredicateFactory` 类 继承 `AbstractRoutePredicateFactory`  

* 在 `TimeBetweenRoutePredicateFactory` 编写静态内部类 `Config` 

    ```java
    @Component
    public class TimeBetweenRoutePredicateFactory extends AbstractRoutePredicateFactory<TimeBetweenRoutePredicateFactory.Config> {
    
        private static final String START_KEY = "start";
        private static final String END_KEY = "end";
        
        public TimeBetweenRoutePredicateFactory() {
            super(TimeBetweenRoutePredicateFactory.Config.class);
        }
    
        @Override
        public Predicate<ServerWebExchange> apply(Config config) {
            LocalTime start = config.getStart();
            LocalTime end = config.getEnd();
            return serverWebExchange -> {
                LocalTime now = LocalTime.now();
                return now.isAfter(start) && now.isBefore(end);
            };
        }
    
        /**
         * 设置配置类与配置文件的关系
         */
        @Override
        public List<String> shortcutFieldOrder() {
            /*
             * 例如我们的配置项是：TimeBetween=上午6:00, 下午1:00
             * 那么按照顺序，start对应的是上午6:00；end对应的是下午1:00
             */
            return Arrays.asList(START_KEY, END_KEY);
        }
    
        static class Config{
            private LocalTime start;
            private LocalTime end;
    
            LocalTime getStart() {
                return start;
            }
    
            void setStart(LocalTime start) {
                this.start = start;
            }
    
            LocalTime getEnd() {
                return end;
            }
    
            void setEnd(LocalTime end) {
                this.end = end;
            }
        }
    }
    ```

* 配置文件

  ```yaml
  spring:
    cloud:
      gateway:
        routes:
          - id: customizer_predicate
            uri: http://localhost:8080/say/
            predicates:
              - TimeBetween=上午6:00,下午1:00
  ```

* 自定义 `Predicate Fatory` 类时，按照 `Spring Cloud Stream` 的约定，类名须为 “ `Predicate` 工厂名（本文例中：`TimeBetween` ）” + `RoutePredicateFactory` 

* 时间格式不是随便配置的，而是Spring Cloud Gateway的默认时间格式



到此为止就实现了一个自定义 `Predicate Fatory` ，若此时不在允许的访问时间段内，访问就会报404，访问：`http://localhost:8088/say` ，结果如下图所示：

![1595860335971](http://123.57.45.66/images/autu_blog/micro_service/springcloud/2020-07-26-spring-cloud-gateway/1595860335971.png)



### 自定义 Filter Factory

需求：记录访问日志

* 自定义 `Filter Factory` 类

  ```java
  @Component
  public class LogCustomizerGatewayFilterFactory extends AbstractGatewayFilterFactory<LogCustomizerGatewayFilterFactory.Config> {
  
      private Logger logger= LoggerFactory.getLogger(LogCustomizerGatewayFilterFactory.class);
  
      private static final String NAME_KEY = "name";
  
      public LogCustomizerGatewayFilterFactory() {
          super(Config.class);
      }
  
      @Override
      public List<String> shortcutFieldOrder() {
          return Arrays.asList(NAME_KEY);
      }
  
      @Override
      public GatewayFilter apply(Config config) {
          //Filter pre  post
          return ((exchange,chain)->{
              logger.info("[pre] Filter Request, name:"+config.getName());
              //TODO
              return chain.filter(exchange).then(Mono.fromRunnable(()->{
                  //TODO
                  logger.info("[post]: Response Filter");
              }));
          });
      }
  
      static class Config{
          private String name;
  
          String getName() {
              return name;
          }
  
          void setName(String name) {
              this.name = name;
          }
      }
  }
  ```

* 添加相关配置

  ```yaml
  spring:
    cloud:
      gateway:
        routes:
          - id: log_customizer
            uri: http://localhost:8080/say/
            predicates:
              - Method=GET
            filters:
              - LogCustomizer=Hello Log Customizer
  ```

* 访问：`http://localhost:8088/say` ，`spring-cloud-gateway-sample` 产生如下日志：

  ```verilog
  2020-07-27 22:45:54.606  INFO 11172 --- [ctor-http-nio-3] .a.e.s.LogCustomizerGatewayFilterFactory : [pre] Filter Request, name:Hello Log Customizer
  2020-07-27 22:45:54.776  INFO 11172 --- [ctor-http-nio-4] .a.e.s.LogCustomizerGatewayFilterFactory : [post]: Response Filter
  ```

  ![1595861334026](http://123.57.45.66/images/autu_blog/micro_service/springcloud/2020-07-26-spring-cloud-gateway/1595861334026.png)

