---
layout: post
title: "Jndi+Tomcat配置数据源"
subtitle: "安全"
author: "Autu"
header-style: text
tags:
  - 项目
  - JNDI
---


#### 背景
在项目开发完成后，在服务器部署时，为了保证数据库连接信息的安全性，我们不能再将数据库配置信息放在properties文件中。因此，我们采用JNDI来将数据库配置信息放入服务器中。

服务器我们以tomcat7为例，来进行详细概述。

#### 实现
JNDI（Java Naming and Directory Interface,Java命名和目录接口）

在<code>Tomcat 4.1.27</code>之后，在服务器上就直接增加了数据源的配置选项，直接在服务器上配置好数据源连接池即可。在J2EE服务器上保存着一个数据库的多个连接。每一个连接通过<code>DataSource</code>可以找到。<code>DataSource</code>被绑定在了JNDI树上（为每一<code>DataSource</code>提供一个名字）客户端通过名称找到在JNDI树上绑定的<code>DataSource</code>，再由<code>DataSource</code>找到一个连接。如下图所示： 

![](/img/project/JNDI_01.png)

每个数据源连接都在JNDI中存在一个唯一的<code>name</code>,当我们连接数据源请求到来时，会到服务器的配置文件中寻找该<code>name</code>对应的<code>Resource</code>来解析数据库配置

在Tomcat的context.xml中<Context></Context>标签内部加入如下配置

```xml
<Resource name="jdbc/localTest" 
				auth="Container" 
				type="javax.sql.DataSource" 
				maxActive="100" 
				maxIdle="30" 
				maxWait="10000" 
				username="scott" 
				password="scott" 
				driverClassName="oracle.jdbc.driver.OracleDriver" 
				url="jdbc:oracle:thin:@127.0.0.1:1521:orcl11" 
	/> 
	
    <Resource name="jdbc/addrlist" 
				auth="Container" 
				type="javax.sql.DataSource" 
				maxActive="100" 
				maxIdle="30" 
				maxWait="10000" 
				username="uum" 
				password="uum" 
				driverClassName="oracle.jdbc.driver.OracleDriver" 
				url="jdbc:oracle:thin:@10.1.239.200:1521:testdb" 
	/> 
```

在applicationContext-dataSource.xml中配置JNDI


```xml
    <!-- 配置数据源 -->
	<jee:jndi-lookup id="dataSource" jndi-name="jdbc/addrlist" resource-ref="true" lookup-on-startup="false"
		proxy-interface="javax.sql.DataSource" />
		
	<!-- 本地测试 -->
	<jee:jndi-lookup id="dataSource" jndi-name="jdbc/localTest" resource-ref="true" lookup-on-startup="false"
		proxy-interface="javax.sql.DataSource" /> 
```


