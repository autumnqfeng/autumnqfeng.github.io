---
layout: post
title: "Mybatis运行原理"
subtitle: "原理分析"
date:       2019-08-07 12:00:00
author: "Autu"
header-img: "img/post-bg-rwd.jpg"
header-mask: 0.3
catalog:    true
tags:
  - Mybatis
  - 原理
  - 源码
---

## 1. mybatis的基本构成

要想了解mybatis的基本构成，首先我们先了解一下mybatis的核心组件。

- SqlSessionFactoryBuilder（构造器）：它会根据配置信息或者代码来生成SQLSessionFactory（工厂接口）。
- SQLSessionFactory：依靠工厂来生成SqlSession（会话）。
- SQLSession：是一个既可以发送sql去执行并返回结果，也可以获取mapper的接口
- SQL Mapper：它是mybatis新设计的组件，它是由一个Java接口文件和xml文件（或注解）构成的，需要给出对应的SQL和映射规则。他负责发送SQL去执行，并返回结果。

用一张图来表示他们之间的关联：

![1570759523924](/img/2019-08-07-mybatis-run-theory/1570759523924.png)

### 1.1 构建SqlSessionFactory

每个mybatis应用都是以SqlSessionFactory的实例为中心的。该实例可以通过SqlSessionFactoryBuilder获得。但我们需要注意的是SqlSessionFactory是一个工厂接口而不是实现类，它的任务是创建SqlSession。

mybatis提供了2种模式去创建SqlSessionFactory：

- 一种是XML配置方式（推荐使用此方式）
- 另一种是代码的方式

Configuration类的全限定名为org.apache.ibatis.session.Configuration，它在mybatis中将以一个Configuration类对象的形式存在，该对象将存在于整个mybatis的生命周期中，以便重复读取和运用。

解析一次XML配置文件保存到Configuration类对象中，即<code>内存</code>中，方便读取配置信息，性能高。

每个数据库只对应一个SqlSessionFactory，因此SqlSessionFactory为<code>单例模式</code>。

下面我们来看一下SqlSessionFactory的构建关系图：

![1570761480512](/img/2019-08-07-mybatis-run-theory/1570761480512.png)

在mybatis中提供了两个SqlSessionFactory的实现类，DefaultSqlSessionFactory和SqlSessionManager。不过SqlSessionManager目前还没有使用，mybatis中目前使用的是DefaultSqlSessionFactory。

### 1.2 创建SqlSession

SqlSession是一个接口类，它扮演着门面的作用，而真正干活的是Executor接口。

我们可以将SqlSession比作公司的美女客服，Executor比作公司的工程师。假设我是客户找你们公司干活，我只需要告诉前台的美女客服（SqlSession）我要什么信息（参数），要做什么东西，过段时间，她会将结果给我。在这个过程中，作为用户所关心的是：

1. 要给美女客服（SqlSession）什么信息（功能和参数）。
2. 美女客服会返回什么结果（Result）。

而我不关心工程师（Executor）是怎么为我工作的，只要前台告诉工程师，工程师就知道如何为我工作。



SqlSession的用途有两种：

1. 获取映射器，让映射器通过命名空间和方法名称找到对应的SQL，发送给数据库执行后返回结果。
2. 直接通过命名信息去执行SQL返回结果，这是ibatis版本留下的方式。在SqlSession层我们可以通过update、insert、select、delete等方法，带上SQL的id来操作在xml中配置好的SQL，从而完成我们的工作；与此同时它也支持事务，通过commit、rollback方法提交或回滚事务。

### 1.3 映射器

映射器是由Java接口和XML文件（或注解）共同组成的，他的作用如下：

- 定义参数类型
- 描述缓存
- 描述SQL语句
- 定义查询结果和POJO的映射关系

一个映射器的实现方式有2种，一种是通过xml方式去实现，另外一种就是通过代码方式去实现。推荐使用xml方式。

#### 1.3.1 xml配置文件的方式实现Mapper

使用xml文件配置是mybatis实现Mapper的首选方式。它是由一个java接口和一个xml文件构成。让我们看看它是如何实现的。

第一步，给出java接口，代码如下：

```java
package com.learn.chapter2.mapper;
import com.learn.chapter2.po.Role;

public interface RoleMapper{
	public Role getRole(Long id);
}
```

这里我们定义了一个接口，它有一个方法getRole，通过角色编号找到角色对象。

第二步，给出一个映射xml文件

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
	PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
	"http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.learn.chapter2.mapper.RoleMapper">
	<select id="getRole" parameterType="long" resultType="role">
		select id, role_name as roleName, note from t_role where id = #{id}
	</select>
</mapper>
```

描述一下上面的xml做了什么

- 这个文件是我们在mybatis-config.xml中配置了的，所以mybatis会读取这个文件，生成映射器。
- 定义了一个命名空间为com.learn.chapter2.mapper.RoleMapper的SQL Mapper，这个命名空间和我们定义的接口的全限定名是一致的
- 用一个select元素定义了一个查询SQL，id为getRole，和接口方法是一致的，而parameterType表示我们传递给这条SQL的是一个java.lang.Long型参数；而resultType则定义需要返回的数据类型，这里的role是注册的com.learn.chapter2.po.Role的别名。

用SqlSession获取Mapper

```java
// 获取映射器Mapper
RoleMapper roleMapper = sqlSession.getMapper(RoleMapper.class);
Role role = roleMapper.getRole(1L);	// 执行方法
System.out.println(role.getRoleName());	// 打印角色名称
```

## 2. 生命周期

正确理解SqlSessionFactoryBuilder、SqlSessionFactory、SqlSession和Mapper的生命周期，对于mybatis应用的正确性和高性能是极其重要的。

### 2.1 SqlSessionFactoryBuilder

SqlSessionFactoryBuilder是利用xml或Java编码获得资源来构建SqlSessionFactory的，通过它可以构建多个SqlSessionFactory。

它的作用就是一个构建器，一旦我们构建了SqlSessionFactory，它的作用就已经完结，失去了存在的意义，这时我们就应该废弃并回收它。

所以它的生命周期只存在于<code>方法的局部</code>，它的作用就是生成SqlSessionFactory对象。

### 2.2 SqlSessionFactory

SqlSessionFactory的作用是创建SqlSession，而SqlSession就是一个会话，相当于jdbc中的Connection对象。

每次应用程序需要访问数据库，我们就要通过SqlSessionFactory创建SqlSession，所以SqlSessionFactory应该在<code>mybatis应用的整个生命周期中</code>。

如果我们多次创建同一个数据库的SqlSessionFactory，则每次创建SqlSessionFactory会打开更多的数据库连接（Connection）资源。

因此SqlSessionFactory的责任是唯一的，就是创建SqlSession，所以采用<code>单例模式</code>，使得每一个数据库只对应一个SqlSessionFactory。

### 2.3 SqlSession

SqlSession是一个会话，相当于JDBC的一个Connection对象，它的生命周期应该是在<code>请求数据库处理事务的过程中</code>。

它是一个线程不安全的对象，在涉及多线程的时候要特别当心，操作数据库需要注意其隔离级别，数据库锁等高级特性。

每次创建都应该<code>及时关闭</code>。

### 2.4 Mapper

Mapper是一个接口，而没有任何实现类，它的作用是发送SQL，然后返回我们需要的结果，或者执行SQL从而修改数据库的数据，因此他应该在一个SqlSession事务方法之内，是一个方法级别的东西。

它就如同JDBC中一条SQL语句的执行，它最大的范围和SqlSession是相同的。

有了上面的描述，我们已经清楚了mybatis的生命周期。

![1570771922793](/img/2019-08-07-mybatis-run-theory/1570771922793.png)



## 3. 构建SqlSessionFactory过程

SqlSessionFactory是mybatis的核心类之一，其最重要的功能就是提供创建mybatis的核心接口SqlSession，所以我们需要先创建SqlSessionFactory，为此我们需要提供配置文件和相关参数。

而mybatis是一个复杂的系统，采用构造模式去创建SqlSessionFactory，我们可以通过SqlSessionFactoryBuilder去构建。构建分为两步。

![1570773105148](/img/2019-08-07-mybatis-run-theory/1570773105148.png)

第一步，通过org.apache.ibatis.builder.xml.XMLConfigBuilder解析配置的XML文件，读出配置参数，并将其存入Configuration类中。<code>注意：mybatis几乎所有配置都是存在这里</code>。

第二步，使用Configuration对象去创建SqlSessionFactory。

mybatis中SqlSessionFactory是一个接口而不是实现类，一般我们使用它的默认实现类DefaultSqlSessionFactory



> 总结：这种创建的方式是一种Builder模式。对于复杂的对象而言，直接使用构造方法构建是有困难的，这会导致大量的逻辑放在构造方法中。为了降低其复杂性，这时使用一个参数类总领全局。
>
> 例如：Configuration类，然后分步构建，DefaultSqlSessionFactory类，就可以构建一个复杂的对象。



## 4. SqlSession运行过程

SqlSession是一个接口，使用它并不复杂。SqlSession给出了查询、插入、更新、删除的方法，在旧版本的mybatis或ibatis中常常使用这些接口方法

在新版的mybatis中我们建议使用Mapper，所以它就是mybatis最为常用和重要的接口之一

### 4.1 映射器的动态代理

Mapper映射是通过动态代理来实现的，首先我们看一下MapperProxyFactory.java的实现。

```java
public class MapperProxyFactory<T> {
	
	......
	
	@SuppressWarnings("unchecked")
	protected T newInstance(MapperProxy<T> mapperProxy) {
		return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
	}
	// 完成SqlSession和接口类的绑定
	public T newInstance(SqlSession sqlSession) {
		final MapperProxy<T> mapperProxy = new MapperProxy<T>(sqlSession, mapperInterface, methodCache);
		return newInstance(mapperProxy);
	}

}
```

这里我们可以看到动态代理对接口的绑定，它的作用就是生成动态代理对象（占位）。而代理的方法则被放到了MapperProxy类中。

接下来我们看一下MapperProxy的源码

```java
public class MapperProxy<T> implements InvocationHandler, Serializable {
	
	......

	@Override
	public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
		if(Object.class.equals(method.getDeclaringClass())) {
			try {
				return method.invoke(this, args);
			} catch (Throwable t) {
				throw ExceptionUtil.unwrapThrowable(t);
			}
		}
		final MapperMethod mapperMethod = cacheMapperMethod(method);
		return mapperMethod.executu(sqlSession, args);
	}

	......
}
```

上面运行了invoke方法。一旦mapper是一个代理对象，那么它就会运行到invoke方法里面，invoke首先判断它是否是一个类，显然这里Mapper是一个接口不是类，所以判定失败。

那么就会生成MapperMethod对象，它是通过cacheMapperMethod方法对其初始化的，然后执行execute方法，把SqlSession和当前运行的参数传递进去。

### 4.2 SqlSession下的四大对象

我们已经知道了映射器其实就是一个动态代理对象，显然我们通过类名和方法名字就可以匹配到我们配置的SQL，我们不需要关心这些细节，我们关心的是设计框架。

mapper执行的过程是通过Executor、StatementHandler、ParameterHandler和ResultHandler来完成数据库操作和结果的返回的。

- Executor代表执行器，由它来调度StatementHandler、ParameterHandler、ResultHandler等来执行对应的SQL。
- StatementHandler的作用是使用数据库的Statement（PreparedStatement）执行操作，它是四大对象的核心，起到承上启下的作用。
- ParameterHandler用于SQL对参数的处理。
- ResultHandler是进行最后数据集（ResultSet）的封装返回处理的。

下面我们逐一分析这四个对象的生成和运作原理

#### 4.2.1 执行器

执行器（Executor）起到了至关重要的作用。它是一个真正执行Java和数据库交互的东西。

在mybatis中存在三种执行器。我们可以在mybatis的配置文件中进行选择。

- SIMPLE，简易执行器，不配置它就是默认执行器
- REUSE，是一种执行器重用预处理语句
- BATCH，执行器重用语句和批量更新，它是针对批量专用的执行器

他们都提供了查询和更新的方法，以及相关的事务方法，下面让我们来看看mybatis如何创建Executor

```java
public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
	executorType = executorType == null ? defaultExecutorType : executorType;
	executorType = executorType == null ? ExecutorType.SIMPLE : executorType;
	Executor executor;
	if(ExecutorType.BATCH == executorType) {
		executor = new BatchExecutor(this, transaction);
	} else if(ExecutorType.REUSE == executorType) {
		executor = new ReuseExecutor(this, transaction);
	} else {
		executor = new SimpleExecutor(this, transaction);
	}
	if(cacheEnabled) {
		executor = new CachingExecutor(executor);
	}
	executor = (Executor) interceptorChain.pluginAll(executor);
	return executor;
}
```

如同描述的一样，mybatis将根据配置类型去确定你需要创建三种执行器中的哪一种。

不妨看看执行器的内部，现在我们以SIMPLE执行器SimpleExecutor的查询方法为例，看一下源码

```java
public class SimpleExecutor extends BaseExecutor {
	......

	@Override
	public <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
		Statement stmt = null;
		try {
			Configuration configuration = ms.getConfiguration();
			StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, resultHandler, boundSql);
			stmt = prepareStatement(handler, ms.getStatementLog());
			return handler.<E>query(stmt, resultHandler);
		} finally {
			closeStatement(stmt);
		}
	}

	private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
		Statement stmt;
		Connection connection = getConnection(statementLog);
		stmt = handler.prepare(connection);
		handler.parameterize(stmt);
		return stmt;
	}

	......
}
```

显然mybatis根据Cofiguration来构建StatementHandler，然后使用prepareStatement方法，对SQL编译并对参数进行初始化。

我们再看它的实现过程，它调用了StatementHandler的prepare()进行了预编译和基础设置，然后通过StatementHandler的parameterize()来设置参数并执行，resultHandler再组装查询结果返回给调用者来完成一次查询。

这样我们的焦点又转移到了StatementHandler上。

#### 4.2.2 数据库会话器

数据库会话器（StatementHandler）就是专门处理数据库会话的，让我们看看mybatis是如何创建StatementHandler的

```java
public StatementHandler newStatementHandler(Executor executor, 
	MappedStatement mappedStatement, Object parameterObject, 
	RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {

	StatementHandler statementHandler = new RoutingStatementHandler(executor, 
		mappedStatement, parameterObject, rowBounds, resultHandler, boundSql);

	statementHandler = (StatementHandler) interceptorChain.pluginAll(statementHandler);
	return statementHandler;
}
```

很显然创建的真实对象是一个RoutingStatementHandler对象，它实现了接口StatementHandler。

RoutingStatementHandler不是我们真实地服务对象，它是通过适配器模式找到对应的StatementHandler来执行的。

在mybatis中，StatementHandler和Executor一样分为三种：

- SimpleStatementHandler
- PreparedStatementHandler
- CallableStatementHandler

它与4.2.1节讨论的三种执行器相对应



在初始化RoutingStatementHandler对象的时候它会根据上下文环境决定创建哪个StatementHandler对象，我们看看RoutingStatementHandler的源码

```java
public class RoutingStatementHandler implements StatementHandler {

	private final StatementHandler delegate;

	public RoutingStatementHandler(Executor executor, MappedStatement ms, 
                                   Object parameter, RowBounds rowBounds, 
                                   ResultHandler resultHandler, BoundSql boundSql) {

		switch (ms.getStatementType()) {
			case STATEMENT:
				delegate = new SimpleStatementHandler(executor, ms, parameter, 
                                                      rowBounds, resultHandler, 
                                                      boundSql);
				break;
			case PREPARED:
				delegate = new PreparedStatementHandler(executor, ms, parameter, 
                                                        rowBounds, resultHandler, 
                                                        boundSql);
				break;
			case CALLABLE:
				delegate = new CallableStatementHandler(executor, ms, parameter, 
                                                        rowBounds, resultHandler, 
                                                        boundSql);
				break;
			default:
				throw new ExecutorException("Unknown statement type: " 
                                            + ms.getStatementType());
		}
	}
	......
}
```

#### 4.2.3 参数处理器

mybatis是通过参数处理器（ParameterHandler）对预编译语句进行参数设置的。它的作用很明显，那就是完成对预编译参数的设置。

```java
public interface ParameterHandler {
    Object getParameterObject();
    void setParameters(PreparedStatement ps) throws SQLException;
}
```

其中，getParameterObject()方法的作用是返回参数对象，setParameters()方法的作用是设置预编译SQL语句的参数。

#### 4.2.4 结果处理器

有了StatementHandler的描述，我们知道它就是组装结果集返回的。

我们再来看看结果处理器（ResultSetHandler）的接口的定义

```java
public interface ResultSetHandler {
    <E> List<E> handleResultSets(Statement stmt) throws SQLException;
    void handleOutputParameters(CallableStatement cs) throws SQLException;
}
```

其中，handleOutputParameters()方法是处理存储过程输出参数的，暂不必管它。

handleResultSets()方法是包装结果集的。

### 4.3 SqlSession运行总结

<center>SqlSession内部运行图</center>

![1570787982981](/img/2019-08-07-mybatis-run-theory/1570787982981.png)

SqlSession是通过Executor创建StatementHandler来运行的，而StatementHandler要经过下面三步。

- prepared预编译SQL
- parameterize设置参数
- query/update执行SQL

其中，parameterize是调用parameterHandler的方法去设置的，而参数是根据类型处理器typeHandler去处理的。

query/update方法是通过resultHandler进行处理结果的封装。

如果是update的语句，它就返回整数，否则它就通过typeHandler处理结果类型，然后用ObjectFactory提供的规则组装对象，返回给调用者。