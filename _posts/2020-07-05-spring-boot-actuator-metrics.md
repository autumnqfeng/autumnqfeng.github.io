---
layout:     post
title: "Spring Boot Metrics监控之Prometheus&Grafana"
subtitle: "Spring Boot Metrics"
date:       2020-07-05 12:13:00
author:     "Autu"
header-img: "img/post-bg-web.jpg"
header-mask: 0.3
catalog:    true
tags:
  - 分布式
  - 微服务
  - springboot
---

## Spring Boot Metrics监控之Prometheus&Grafana

在第一部分中，我们学习到了`spring-boot-actuator`模块做了什么，如何配置spring boot应用以及如何与各样的actuator endpoints交互。

在这篇文章中，我们将学习spring boot如何整合外部监控系统[Prometheus](https://prometheus.io/)和图表解决方案[Grafana](https://grafana.com/)。

在这篇文章的末尾，我们将在自己本地电脑建立一个Prometheus和Grafana仪表盘，用来可视化监控Spring Boot应用产生的所有metrics。

### Prometheus

Prometheus是一个开源的监控系统，它由以下几个核心组件构成：

- 数据爬虫：根据配置的时间定期通过HTTP抓取metrics数据。
- [time-series](https://en.wikipedia.org/wiki/Time_series) 数据库：存储所有的metrics数据。
- 简单的用户交互接口：可视化、查询和监控所有的metrics。

### Grafana

Grafana使你能够把来自不同数据源，比如：Elasticsearch、Prometheus、Graphite、influxDB等多样的数据以绚丽的图标展示出来。

它也能基于你的metrics数据发出告警。当一个告警状态改变时，它能通过email、slack或者其他途径通知你。

值得注意的是，Prometheus仪表盘也有简单的图标。但是Grafana的图标表现的更好。这也是为什么，在这篇文章中，我们将整合Grafana和Prometheus来可视化metrics数据。

### 增加Micrometer Prometheus Registry到Spring Boot应用

为了整合Prometheus，你需要增加`micrometer-redistry-prometheus`依赖：

```xml
<!-- Micrometer Prometheus registry  -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

一旦你增加了上述的依赖，Spring Boot会自动配置一个[`PrometheusMeterRegistry`](https://github.com/micrometer-metrics/micrometer/blob/master/implementations/micrometer-registry-prometheus/src/main/java/io/micrometer/prometheus/PrometheusMeterRegistry.java)和[`CollectorRegistry`](https://github.com/prometheus/client_java/blob/master/simpleclient/src/main/java/io/prometheus/client/CollectorRegistry.java)来收集和输出格式化的metrics数据，使得Prometheus服务器可以爬取。

所有应用的metrics数据是根据一个叫`/prometheus` 的endpoint来设置是否可用。Prometheus服务器可以周期性的爬取这个endpoint来取metrics数据。

### 解析Spring Boot Actuator的`/prometheus` endpoint

首先，你可以通过actuator endpoint-discovery页面（<http://localhost:8080/actuator>）来看一下prometheus endpoint。

```json
"prometheus": {
    "href": "http://127.0.0.1:8080/actuator/prometheus",
    "templated": false
}
```

`rometheus` endpoint暴露了格式化的metrics数据给Prometheus服务器。你可以通过`prometheus` endpoint(<http://localhost:8080/actuator/prometheus>)看到被暴露的metrics数据:

```plain
# HELP system_cpu_count The number of processors available to the Java virtual machine
# TYPE system_cpu_count gauge
system_cpu_count 4.0
# HELP tomcat_sessions_expired_sessions_total  
# TYPE tomcat_sessions_expired_sessions_total counter
tomcat_sessions_expired_sessions_total 0.0
# HELP jvm_classes_loaded_classes The number of classes that are currently loaded in the Java virtual machine
# TYPE jvm_classes_loaded_classes gauge
jvm_classes_loaded_classes 7076.0
# HELP jvm_gc_live_data_size_bytes Size of old generation memory pool after a full GC
# TYPE jvm_gc_live_data_size_bytes gauge
jvm_gc_live_data_size_bytes 1.5777464E7
# HELP jvm_buffer_count_buffers An estimate of the number of buffers in the pool
# TYPE jvm_buffer_count_buffers gauge
jvm_buffer_count_buffers{id="direct",} 7.0
jvm_buffer_count_buffers{id="mapped",} 0.0
# HELP tomcat_sessions_alive_max_seconds  
# TYPE tomcat_sessions_alive_max_seconds gauge
tomcat_sessions_alive_max_seconds 0.0
# HELP process_cpu_usage The "recent cpu usage" for the Java Virtual Machine process
# TYPE process_cpu_usage gauge
process_cpu_usage 2.1122726120743928E-4
# HELP jvm_threads_states_threads The current number of threads having NEW state
# TYPE jvm_threads_states_threads gauge
jvm_threads_states_threads{state="runnable",} 10.0
jvm_threads_states_threads{state="blocked",} 0.0
jvm_threads_states_threads{state="waiting",} 12.0
jvm_threads_states_threads{state="timed-waiting",} 3.0
jvm_threads_states_threads{state="new",} 0.0
jvm_threads_states_threads{state="terminated",} 0.0

......
```

### 使用Docker下载和运行Prometheus

#### 下载Prometheus

你可以使用`docker pull`命令来下载Prometheus docker image。

```shell
$ docker pull prom/prometheus
```

一旦这个images被下载下来，你就可以使用`docker images` 命令来查看本地的image列表：

```shell
$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
prom/prometheus     latest              9f345bfa8fef        8 days ago          142MB
```

#### Prometheus配置（prometheus.yml）

接下来，我们需要配置Prometheus来抓取Spring Boot Actuator的`/prometheus` endpoint中的metrics数据。

创建一个prometheus.yml的文件，填入以下内容：

```yml
# my global config
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'
    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.
    static_configs:
    - targets: ['127.0.0.1:9090']

  - job_name: 'spring-actuator'
    metrics_path: '/actuator/prometheus'
    scrape_interval: 5s
    static_configs:
    - targets: ['HOST_IP:8080']
```

在Prometheus文档中，上面的配置文件是[basic configuration file](https://prometheus.io/docs/prometheus/latest/getting_started/#configuring-prometheus-to-monitor-itself)的扩展。

上面中比较重要的配置项是`spring-actuator` job中的`scrape_configs`选项。

`metrics_path`是Actuator中`prometheus` endpoint中的路径。`targes`包含了Spring Boot应用的`HOST`和`PORT`。

请确保替换`HOST_IP`为你Spring Boot应用运行的电脑的IP地址。值得注意的是，`localhost`将不起作用，因为我们将从docker container中连接HOST机器。你必须设置网络IP地址。

#### 使用Docker运行Prometheus

最后，让我们在Docker中运行Prometheus。使用以下命令来启动一个Prometheus服务器。

```shell
$ docker run -d --name=prometheus -p 9090:9090 -v <PATH_TO_prometheus.yml_FILE>:/etc/prometheus/prometheus.yml prom/prometheus --config.file=/etc/prometheus/prometheus.yml
```

请确保替换`<PATH_TO_prometheus.yml_FILE>`为你在上面创建的Prometheus配置文件的保存的路径。

在运行上述命令之后，docker将在container中启动一个Prometheus服务器。你可以通过以下命令看到所有的container：

```shell
$ docker ps -a
CONTAINER ID        IMAGE                 COMMAND                  CREATED             STATUS              PORTS                                                                                        NAMES
3cad520251b6        prom/prometheus       "/bin/prometheus --c…"   About an hour ago   Up About an hour    0.0.0.0:9090->9090/tcp                                                                       prometheus
```

#### 在Prometheus仪表盘中可视化Spring Boot Metrics

你可以通过访问http://localhost:9090访问Prometheus仪表盘。你可以通过Prometheus查询表达式来查询metrics。

下面是一些例子：

- 系统CPU使用

  ![1593919551357](http://123.57.45.66/images/autu_blog/micro_service/springboot/2020-07-05-spring-boot-actuator-metrics/1593919551357.png)

- API延迟响应

  ![1593919691025](http://123.57.45.66/images/autu_blog/micro_service/springboot/2020-07-05-spring-boot-actuator-metrics/1593919691025.png)

你可以从Prometheus官方文档中学习更多的 [`Prometheus Query Expressions`](https://prometheus.io/docs/introduction/first_steps/#using-the-expression-browser)。

### 使用Docker下载和运行Grafana

#### 下载Grafana

使用以下命令可以使Docker下载和运行Grafana：

```shell
$ docker run -d --name=grafana -p 3000:3000 grafana/grafana
```

上述命令将在Docker Container中开启一个Grafana，并且使用3000端口在主机上提供服务。

你可以使用`docker ps -a`来查看Docker container列表：

```shell
$ docker ps -a
CONTAINER ID        IMAGE                 COMMAND                  CREATED             STATUS              PORTS                                                                                        NAMES
fe5885671628        grafana/grafana       "/run.sh"                14 seconds ago      Up 12 seconds       0.0.0.0:3000->3000/tcp                                                                       grafana
88e8e0effe6c        prom/prometheus       "/bin/prometheus --c…"   18 minutes ago      Up 18 minutes       0.0.0.0:9090->9090/tcp                                                                       prometheus
```

你可以访问[http://localhost:3000](http://localhost:3000/)，并且使用默认的账户名(admin)密码(admin)来登录Grafana

#### 配置Grafana导入Prometheus中的metrics数据

通过以下几步导入Prometheus中的metrics数据并在Grafana上可视化。

##### 在Grafana上添加数据源

![1593920356152](http://123.57.45.66/images/autu_blog/micro_service/springboot/2020-07-05-spring-boot-actuator-metrics/1593920356152.png)

##### 建立一个仪表盘图表

![1593920637535](http://123.57.45.66/images/autu_blog/micro_service/springboot/2020-07-05-spring-boot-actuator-metrics/1593920637535.png)

##### 添加一个Prometheus查询

![1593920708865](http://123.57.45.66/images/autu_blog/micro_service/springboot/2020-07-05-spring-boot-actuator-metrics/1593920708865.png)

##### 默认可视化

Grafana的面板配置过程还是比较繁琐的，如果我们不想自己去配置，那我们可以去Grafana官网上去
下载一个dashboard。

- 模板地址：https://grafana.com/dashboards
- 在搜索框中搜索 Spring Boot 会检索出相关的模板，选择一个自己喜欢的。
- 这里可以采用: https://grafana.com/grafana/dashboards/10280 这个，看起来比较清晰

![1593921340778](http://123.57.45.66/images/autu_blog/micro_service/springboot/2020-07-05-spring-boot-actuator-metrics/1593921340778.png)



阅读第一部分

### 翻译源

- [Spring Boot Actuator: Health check, Auditing, Metrics gathering and Monitoring](https://www.callicoder.com/spring-boot-actuator/)