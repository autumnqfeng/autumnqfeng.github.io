---
layout:     post
title: "spring cloud config配置中心原理分析"
subtitle: "源码"
date:       2020-07-15 12:13:00
author:     "Autu"
header-img: "img/post-bg-web.jpg"
header-mask: 0.3
catalog:    true
tags:
  - 分布式
  - 微服务
  - springcloud
---

## spring cloud config配置中心原理分析

### 前言

一般我们将配置放在git或gitee上，因此我们也是基于Git方式讲解spring cloud配置中心原理。

首先，我们先看几个问题，然后带着问题去探索其中的究竟：

* 对于使用过springcloud config的同学都知道，spring cloud分为config client和config server，那么spring配置中心是如何通过client和server获取到git中的配置信息呢？

* git中配置文件更新后，config server端马上可以获取到更新后的内容，而config client端不能马上获取更新后的内容，对于配置了bus的项目，需要触发 http://localhost:8001/actuator/bus-refresh 才可以使得客户端更新配置。这又是为什么呢？



### Config Client 配置加载过程

#### SpringApplication.run

在spring boot项目启动时，有一个`prepareContext`的方法，它会回调所有实现了`ApplicationContextInitializer `的实例，来做一些初始化工作。

```java
public ConfigurableApplicationContext run(String... args) {
 //省略代码...
    prepareContext(context, environment, listeners, applicationArguments, printedBanner);
 //省略代码
  return context;
}
```

#### PropertySourceBootstrapConfiguration.initialize

`PropertySourceBootstrapConfiguration`  实现了  `ApplicationContextInitializer`  接口，其目的就是在应用程序上下文初始化的时候做一些额外的操作。

根据默认的 ` AnnotationAwareOrderComparator ` 排序规则对 `propertySourceLocators` 数组进行排序

获取运行的环境上下文 `ConfigurableEnvironment` 

遍历 `propertySourceLocators时` 

- 调用 `locate` 方法，传入获取的上下文 `environment` 
- 将 `source` 添加到 `PropertySource` 的链表中
- 设置 `source` 是否为空的标识标量 `empty` 
-  ` source` 不为空的情况，才会设置到 `environment` 中
- 返回 `Environment` 的可变形式，可进行的操作如 `addFirst` 、`addLast` 
- 移除 `propertySources` 中的 `bootstrapProperties` 
- 根据 `config server` 覆写的规则，设置 `propertySources` 
- 处理多个 `active profiles` 的配置信息

```java
@Override
public void initialize(ConfigurableApplicationContext applicationContext) {
    List<PropertySource<?>> composite = new ArrayList<>();
    // 对propertySourceLocators数组进行排序，根据默认的AnnotationAwareOrderComparator
    AnnotationAwareOrderComparator.sort(this.propertySourceLocators);
    boolean empty = true;
    // 获取运行的环境上下文
    ConfigurableEnvironment environment = applicationContext.getEnvironment();
    for (PropertySourceLocator locator : this.propertySourceLocators) {
        // 回调所有实现PropertySourceLocator接口实例的locate方法
        Collection<PropertySource<?>> source = locator.locateCollection(environment);
        if (source == null || source.size() == 0) {
            continue;
        }
        List<PropertySource<?>> sourceList = new ArrayList<>();
        for (PropertySource<?> p : source) {
            sourceList.add(new BootstrapPropertySource<>(p));
        }
        logger.info("Located property source: " + sourceList);
        // 将source添加到数组
        composite.addAll(sourceList);
        // 表示propertysource不为空
        empty = false;
    }
    // 只有propertysource不为空的情况，才会设置到environment中
    if (!empty) {
        MutablePropertySources propertySources = environment.getPropertySources();
        String logConfig = environment.resolvePlaceholders("${logging.config:}");
        LogFile logFile = LogFile.get(environment);
        for (PropertySource<?> p : environment.getPropertySources()) {
            if (p.getName().startsWith(BOOTSTRAP_PROPERTY_SOURCE_NAME)) {
                propertySources.remove(p.getName());
            }
        }
        insertPropertySources(propertySources, composite);
        reinitializeLoggingSystem(environment, logConfig, logFile);
        setLogLevels(applicationContext, environment);
        handleIncludedProfiles(environment);
    }
}
```

#### PropertySourceLoader.locateCollection

这个方法会调用子类的 `locate` 方法，来获得一个 `PropertySource` ，然后将 `PropertySource` 集合返回。

接着它会调用 `ConfigServicePropertySourceLocator` 的 `locate` 方法。

```java
static Collection<PropertySource<?>> locateCollection(PropertySourceLocator locator,
                                                      Environment environment) {
    // 调用locate方法
    PropertySource<?> propertySource = locator.locate(environment);
    if (propertySource == null) {
        return Collections.emptyList();
    }
    if (CompositePropertySource.class.isInstance(propertySource)) {
        Collection<PropertySource<?>> sources = ((CompositePropertySource) propertySource)
            .getPropertySources();
        List<PropertySource<?>> filteredSources = new ArrayList<>();
        for (PropertySource<?> p : sources) {
            if (p != null) {
                filteredSources.add(p);
            }
        }
        return filteredSources;
    }
    else {
        return Arrays.asList(propertySource);
    }
}
```

#### ConfigServicePropertySourceLocator.locate

这个就是 `Config Client` 的关键实现了，它会通过 `RestTemplate` **调用一个远程地址**获得配置信息，
`getRemoteEnvironment` 。

然后把这个配置 `PropertySources` ，然后将这个信息包装成一个 `OriginTrackedMapPropertySource` ，设置到 `Composite` 中。

```java
@Override
@Retryable(interceptor = "configServerRetryInterceptor")
public org.springframework.core.env.PropertySource<?> locate(
    org.springframework.core.env.Environment environment) {
    ConfigClientProperties properties = this.defaultProperties.override(environment);
    CompositePropertySource composite = new OriginTrackedCompositePropertySource(
        "configService");
    RestTemplate restTemplate = this.restTemplate == null
        ? getSecureRestTemplate(properties) : this.restTemplate;
    Exception error = null;
    String errorBody = null;
    try {
        String[] labels = new String[] { "" };
        if (StringUtils.hasText(properties.getLabel())) {
            labels = StringUtils
                .commaDelimitedListToStringArray(properties.getLabel());
        }
        String state = ConfigClientStateHolder.getState();
        // Try all the labels until one works
        for (String label : labels) {
            // 获取远程Environment。通过该方法获取config server的配置
            Environment result = getRemoteEnvironment(restTemplate, properties,
                                                      label.trim(), state);
            if (result != null) {
                log(result);

                // result.getPropertySources() can be null if using xml
                if (result.getPropertySources() != null) {
                    for (PropertySource source : result.getPropertySources()) {
                        @SuppressWarnings("unchecked")
                        Map<String, Object> map = translateOrigins(source.getName(),
                                                                   (Map<String, Object>) source.getSource());
                        composite.addPropertySource(
                            new OriginTrackedMapPropertySource(source.getName(),
                                                               map));
                    }
                }

                if (StringUtils.hasText(result.getState())
                    || StringUtils.hasText(result.getVersion())) {
                    HashMap<String, Object> map = new HashMap<>();
                    putValue(map, "config.client.state", result.getState());
                    putValue(map, "config.client.version", result.getVersion());
                    composite.addFirstPropertySource(
                        new MapPropertySource("configClient", map));
                }
                return composite;
            }
        }
        errorBody = String.format("None of labels %s found", Arrays.toString(labels));
    }
    catch (HttpServerErrorException e) {
        error = e;
        if (MediaType.APPLICATION_JSON
            .includes(e.getResponseHeaders().getContentType())) {
            errorBody = e.getResponseBodyAsString();
        }
    }
    catch (Exception e) {
        error = e;
    }
    if (properties.isFailFast()) {
        throw new IllegalStateException(
            "Could not locate PropertySource and the fail fast property is set, failing"
            + (errorBody == null ? "" : ": " + errorBody),
            error);
    }
    logger.warn("Could not locate PropertySource: "
                + (error != null ? error.getMessage() : errorBody));
    return null;

}
```

#### ConfigServicePropertySourceLocator.getRemoteEnvironment

在此方法中发送http请求，获取config server中的配置信息，并返回 `Environment` 。

```java
private Environment getRemoteEnvironment(RestTemplate restTemplate,
                                         ConfigClientProperties properties, String label, String state) {
    String path = "/{name}/{profile}";
    // 获取服务名称
    String name = properties.getName();
    String profile = properties.getProfile();
    String token = properties.getToken();
    int noOfUrls = properties.getUri().length;
    if (noOfUrls > 1) {
        logger.info("Multiple Config Server Urls found listed.");
    }

    Object[] args = new String[] { name, profile };
    if (StringUtils.hasText(label)) {
        // workaround for Spring MVC matching / in paths
        label = Environment.denormalize(label);
        args = new String[] { name, profile, label };
        path = path + "/{label}";
    }
    ResponseEntity<Environment> response = null;

    for (int i = 0; i < noOfUrls; i++) {
        Credentials credentials = properties.getCredentials(i);
        // 获取URI
        String uri = credentials.getUri();
        String username = credentials.getUsername();
        String password = credentials.getPassword();

        logger.info("Fetching config from server at : " + uri);

        try {
            HttpHeaders headers = new HttpHeaders();
            headers.setAccept(
                Collections.singletonList(MediaType.parseMediaType(V2_JSON)));
            addAuthorizationToken(properties, headers, username, password);
            if (StringUtils.hasText(token)) {
                headers.add(TOKEN_HEADER, token);
            }
            if (StringUtils.hasText(state) && properties.isSendState()) {
                headers.add(STATE_HEADER, state);
            }

            final HttpEntity<Void> entity = new HttpEntity<>((Void) null, headers);
            // 发送http请求
            response = restTemplate.exchange(uri + path, HttpMethod.GET, entity,
                                             Environment.class, args);
        }
        catch (HttpClientErrorException e) {
            if (e.getStatusCode() != HttpStatus.NOT_FOUND) {
                throw e;
            }
        }
        catch (ResourceAccessException e) {
            logger.info("Connect Timeout Exception on Url - " + uri
                        + ". Will be trying the next url if available");
            if (i == noOfUrls - 1) {
                throw e;
            }
            else {
                continue;
            }
        }

        if (response == null || response.getStatusCode() != HttpStatus.OK) {
            return null;
        }

        Environment result = response.getBody();
        return result;
    }

    return null;
}
```

#### 小结

* `config client` 端是通过发送 http 请求到 `config server` ，获取 `Environment` 

* 将 `Environment` 中的 `PropertySources` 添加到 client 的 `Environment` 中
* 通过springboot的监听器加载 `PropertySources` 



### Config Server获取配置过程

在config server端我们通过` http://[config-server-ip]:[config-server-port]/` 加以下URI获取`Git`中的配置。

```
/{application}/{profile}[/{label}]
/{application}-{profile}.yml
/{label}/{application}-{profile}.yml
/{application}-{profile}.properties
/{label}/{application}-{profile}.properties
```

#### EnvironmentController

`Spring Cloud Config Server`提供了`EnvironmentController`，这样通过在浏览器访问即可从git中获取配
置信息。

在这个controller中，提供了很多的映射，最终会调用的是 `getEnvironment` 。

```java
@RequestMapping(path = "/{name}/{profiles:.*[^-].*}",
                produces = MediaType.APPLICATION_JSON_VALUE)
public Environment defaultLabel(@PathVariable String name,
                                @PathVariable String profiles) {
    return getEnvironment(name, profiles, null, false);
}

@RequestMapping(path = "/{name}/{profiles:.*[^-].*}",
                produces = EnvironmentMediaType.V2_JSON)
public Environment defaultLabelIncludeOrigin(@PathVariable String name,
                                             @PathVariable String profiles) {
    return getEnvironment(name, profiles, null, true);
}
```

接下来我们看，`getEnvironment`方法：

```java
	public Environment getEnvironment(String name, String profiles, String label,
			boolean includeOrigin) {
		name = Environment.normalize(name);
		label = Environment.normalize(label);
		Environment environment = this.repository.findOne(name, profiles, label,
				includeOrigin);
		if (!this.acceptEmpty
				&& (environment == null || environment.getPropertySources().isEmpty())) {
			throw new EnvironmentNotFoundException("Profile Not found");
		}
		return environment;
	}
```

`this.repository.findOne` ，调用某个repository存储组件来获得环境配置信息进行返回。

repository是一个` EnvironmentRepository` 对象，它有很多实现，其中就包`RedisEnvironmentRepository` 、` JdbcEnvironmentRepository `等。

默认实现是`MultipleJGitEnvironmentRepository` ，表示多个不同地址的git数据源。



#### MultipleJGitEnvironmentRepository.findOne

`MultipleJGitEnvironmentRepository `代理遍历每个 `JGitEnvironmentRepository`

`JGitEnvironmentRepository` 下使用 `NativeEnvironmentRepository `**代理读取本地文件**。

```java
@Override
	public Environment findOne(String application, String profile, String label,
			boolean includeOrigin) {
		// 遍历所有Git源
        for (PatternMatchingJGitEnvironmentRepository repository : this.repos.values()) {
			if (repository.matches(application, profile, label)) {
				for (JGitEnvironmentRepository candidate : getRepositories(repository,
						application, profile, label)) {
					try {
						if (label == null) {
							label = candidate.getDefaultLabel();
						}
						Environment source = candidate.findOne(application, profile,
								label, includeOrigin);
						if (source != null) {
							return source;
						}
					}
					catch (Exception e) {
						if (this.logger.isDebugEnabled()) {
							this.logger.debug(
									"Cannot load configuration from " + candidate.getUri()
											+ ", cause: (" + e.getClass().getSimpleName()
											+ ") " + e.getMessage(),
									e);
						}
						continue;
					}
				}
			}
		}
		JGitEnvironmentRepository candidate = getRepository(this, application, profile,
				label);
		if (label == null) {
			label = candidate.getDefaultLabel();
		}
		if (candidate == this) {
			return super.findOne(application, profile, label, includeOrigin);
		}
		return candidate.findOne(application, profile, label, includeOrigin);
	}
```

#### AbstractScmEnvironmentRepository.findOne

调用抽象类的findOne方法，主要有两个核心逻辑

* 调用`getLocations`从`GIT`远程仓库同步到本地

* 使用 `NativeEnvironmentRepository `委托，来读取本地文件内容

```java
	@Override
	public synchronized Environment findOne(String application, String profile,
			String label, boolean includeOrigin) {
		// 委托NativeEnvironmentRepository读取本地文件内容
        NativeEnvironmentRepository delegate = new NativeEnvironmentRepository(
				getEnvironment(), new NativeEnvironmentProperties());
        // 从Git远程仓库到本地
		Locations locations = getLocations(application, profile, label);
		delegate.setSearchLocations(locations.getLocations());
		Environment result = delegate.findOne(application, profile, "", includeOrigin);
		result.setVersion(locations.getVersion());
		result.setLabel(label);
		return this.cleaner.clean(result, getWorkingDirectory().toURI().toString(),
				getUri());
	}
```

#### JGitEnvironmentRepository.getLocations

```java
@Override
	public synchronized Locations getLocations(String application, String profile,
			String label) {
		if (label == null) {
			label = this.defaultLabel;
		}
        // 刷新Git仓库中配置
		String version = refresh(label);
		return new Locations(application, profile, label, version,
				getSearchLocations(getWorkingDirectory(), application, profile, label));
	}
```

#### JGitEnvironmentRepository.refresh

```java
public String refresh(String label) {
   Git git = createGitClient();
   ...
   checkout(git, label);
   ...
   merge(git, label);
   ...
   resetHard(...)
   ...
}
```

#### 小结

* `controller` 接收http请求，遍历所有Git仓库
* 调用 `Git` 命令，从 `Git` 上 `pull` 配置信息
* 委托 `NativeEnvironmentRepository ` 读取本地配置信息

### 总结

通过上边的源码分析，开始的2个问题，几乎已经很明了了：

* `config server` 端之所以可以获取最新的配置，是因为每次访问都会到 `git` 中拉去配置信息。

* `config client` 需要 `Actuator` 触发 `refresh` 才能更新配置，是因为 client 通过 http 请求获取到的配置信息，在spring boot启动时加载到了 `Environment` 中，只有触发 `refresh` 或**重启应用**才能更新 `Environment` 。

