---
layout:     post
title: "Spring Aop方式添加系统日志"
subtitle: "添加系统日志"
date:       2020-02-25 16:38:00
author:     "Autu"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
  - 6_Cloud
  - CSEC
  - Spring
  - 后端
---

#### 背景介绍

- 功能重构背景：在云盾（H3C）V3.0上线前的一周，由于系统日志功能较为单一，只能记录日志服务器ip配置及重置，并且日志展现的信息较为匮乏。因此该功能在该版本选择屏蔽，操作日志原界面如下：

  

  ![img](/img/6-cloud/Csec/syslog/1582620935501.png)

  

- 原来逻辑：定义 <code>setSysLog(...)</code> 的接口，在配置日志服务器地址端口处调用该接口



> 由于V3.0版本中各功能代码已相对完善，选择改动功能逻辑为大忌，因此我们采用Spring Aop + 注解方式重构该功能。



#### 重构步骤

##### springboot中导入相关依赖

```xml
        <!-- 切面相关依赖 -->
        <dependency>
            <groupId>org.aspectj</groupId>
            <artifactId>aspectjrt</artifactId>
            <version>1.8.10</version>
        </dependency>
        <dependency>
            <groupId>org.aspectj</groupId>
            <artifactId>aspectjtools</artifactId>
            <version>1.8.10</version>
        </dependency>
```

##### 构建实体类

```java
/**
 * @author ***
 * @date 2019/11/14 17:26
 */
@ApiModel(value="系统日志对象",description = "获取系统日志相关信息")
@Document(collection = "log")
@JsonInclude(JsonInclude.Include.NON_NULL)
public class Log implements Serializable {
    private static final long serialVersionUID = -1109721626367874025L;

    @Id
    private String id;
    /** 功能模块 */
    private String moduleName;
    /** 操作名称 */
    private String operateName;
    /** 操作对象 */
    private String operateObj;
    /** 操作详情 */
    private String remark;
    /** 目标方法参数信息 */
    private String targetMethodFields;
    /** 操作结果(success/fail) */
    private String result;
    /** 日志产生时间 */
    private String startTime;
    /** 日志结束时间 */
    private String endTime;
    /** 日志产生的时间戳 */
    private Double timestamp;

    public Log() {
    }

    public Log(String moduleName, String operateName, String operateObj, 
               String remark, String targetMethodFields, String startTime, 
               String endTime, Double timestamp) {
        this.moduleName = moduleName;
        this.operateName = operateName;
        this.operateObj = operateObj;
        this.remark = remark;
        this.targetMethodFields = targetMethodFields;
        this.startTime = startTime;
        this.endTime = endTime;
        this.timestamp = timestamp;
    }
    
    // toString和set、get方法略
}
```

##### 创建注解

- 创建作用在类上的注解：用来记录操作属于哪个功能模块

  ```java
  package com.csec.csos.annotation;
  
  import org.springframework.data.mongodb.core.mapping.Document;
  
  import java.lang.annotation.ElementType;
  import java.lang.annotation.Retention;
  import java.lang.annotation.RetentionPolicy;
  import java.lang.annotation.Target;
  
  /**
   * 日志记录(类注解) 自定义注解
   * @author: ***
   * @date: 2020/2/24 16:23
   */
  @Retention(RetentionPolicy.RUNTIME)
  @Target(ElementType.TYPE)
  @Document
  public @interface LogFunction {
      // 功能模块名称
      String moduleName() default "";
  }
  
  ```

- 创建作用在方法上的注解：记录详细的日志信息

  ```java
  package com.csec.csos.annotation;
  
  import org.springframework.data.mongodb.core.mapping.Document;
  
  import java.lang.annotation.ElementType;
  import java.lang.annotation.Retention;
  import java.lang.annotation.RetentionPolicy;
  import java.lang.annotation.Target;
  
  /**
   * 日志记录(方法注解)
   * @author: ***
   * @date: 2020/2/24 17:39
   */
  @Retention(RetentionPolicy.RUNTIME)
  @Target(ElementType.METHOD)
  @Document
  public @interface LogOperate {
      // 操作名称
      String operateName() default "";
      // 操作对象
      String operateObj() default "";
      // 操作详情
      String remark() default "";
  }
  
  ```

##### 定义切面类

- 切入点：所有service包下带LogOperate注解的方法。

- 前置通知（@Before）：设置开始时间参数

- 环绕通知（@Around）：获取方法返回值

- 后置通知（@After）：对存入日志信息的处理

- 异常通知（@AfterThrowing）：异常处理，并处理日志信息


```java
package com.csec.csos.aspect;

import com.csec.csos.annotation.LogFunction;
import com.csec.csos.annotation.LogOperate;
import com.csec.csos.constant.CodeList;
import com.csec.csos.entity.Log;
import com.csec.csos.entity.ResResult;
import com.csec.csos.util.DateUtil;
import org.aspectj.lang.JoinPoint;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.*;
import org.aspectj.lang.reflect.MethodSignature;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.core.DefaultParameterNameDiscoverer;
import org.springframework.core.ParameterNameDiscoverer;
import org.springframework.data.mongodb.core.MongoTemplate;
import org.springframework.stereotype.Component;

import java.lang.reflect.Method;
import java.util.HashMap;
import java.util.Map;

/**
 * 日志切面类
 * @author: ***
 * @date: 2020/2/24 16:28
 */
@Aspect
@Component
public class LogAspect {
    private static Logger logger = LoggerFactory.getLogger(LogAspect.class);

    @Autowired
    private MongoTemplate mongoTemplate;

    private String startTime;
    private long timestamp;
    private HashMap<String, Object> targetMethodFields;
    private String result;

    @Pointcut("execution(* com.csec.csos.service.*.*(..)) && @annotation(com.csec.csos.annotation.LogOperate)")
    public void pointCut() {}

    @Before("pointCut()")
    public void before(JoinPoint joinPoint) {
        startTime = DateUtil.getDateTime();
        timestamp = System.currentTimeMillis();
        try {
            targetMethodFields = getFieldsMsg(joinPoint);
        } catch (ClassNotFoundException | NoSuchMethodException e) {
            logger.error("切入点处，未匹配到对应的 类或方法");
        }
    }

    @Around("pointCut()")
    public Object around(ProceedingJoinPoint joinPoint) throws Throwable {
        if (joinPoint.proceed() instanceof ResResult) {
            ResResult resResult = (ResResult) joinPoint.proceed();
            if (CodeList.SUCCESS == resResult.getCode()) {
                result = "success";
            } else {
                result = "fail";
            }
            return resResult;
        } else {
            return null;
        }
    }

    @After("pointCut()")
    public void after(JoinPoint joinPoint) {
        Log logInfo = getLogInfo(joinPoint);
        logInfo.setResult(result);
        mongoTemplate.insert(logInfo);
    }

    @AfterThrowing("pointCut()")
    public void exception(JoinPoint joinPoint) {
        Log logInfo = getLogInfo(joinPoint);
        logInfo.setResult("fail");
        mongoTemplate.insert(logInfo);
    }

    private Log getLogInfo(JoinPoint joinPoint) {
        MethodSignature signature = (MethodSignature) joinPoint.getSignature();
        Method method = signature.getMethod();
        LogOperate annotationOperate = method.getAnnotation(LogOperate.class);

        // 获取类上的注解
        LogFunction annotationFunction = joinPoint.getTarget().getClass().getAnnotation(LogFunction.class);

        // TODO:部署vfw逻辑待完善
        String endTime = DateUtil.getDateTime();

        // 遍历参数信息
        StringBuilder targetMethodFieldsBuilder = new StringBuilder();
        for (Map.Entry<String, Object> param : targetMethodFields.entrySet()) {
            targetMethodFieldsBuilder.append(param.getKey());
            targetMethodFieldsBuilder.append("=");
            targetMethodFieldsBuilder.append(param.getValue());
            targetMethodFieldsBuilder.append("；");
        }
        return new Log(annotationFunction.moduleName(),
                annotationOperate.operateName(),
                annotationOperate.operateObj(),
                annotationOperate.remark(),
                targetMethodFieldsBuilder.toString(),
                startTime,
                endTime,
                (double) timestamp);
    }

    /**
     * Integer.class != int.class
     */
    private static HashMap<String, Class> map = new HashMap<String, Class>() {
        private static final long serialVersionUID = 5633590873251228986L;
        {
            put("java.lang.Integer", int.class);
            put("java.lang.Double", double.class);
            put("java.lang.Float", float.class);
            put("java.lang.Long", long.class);
            put("java.lang.Short", short.class);
            put("java.lang.Boolean", boolean.class);
            put("java.lang.Char", char.class);
            put("java.lang.Byte", byte.class);
        }
    };

    /**
     * 获取目标方法中的参数信息
     */
    private HashMap<String, Object> getFieldsMsg(JoinPoint joinPoint) throws 
        ClassNotFoundException, NoSuchMethodException {
        
        String classType = joinPoint.getTarget().getClass().getName();
        String methodName = joinPoint.getSignature().getName();
        Object[] args = joinPoint.getArgs();
        Class<?>[] classes = new Class[args.length];
        for (int k = 0; k < args.length; k++) {
            if (!args[k].getClass().isPrimitive()) {
                //获取的是封装类型而不是基础类型
                String result = args[k].getClass().getName();
                Class s = map.get(result);
                classes[k] = s == null ? args[k].getClass() : s;
            }
        }
        // 获取参数名
        ParameterNameDiscoverer pnd = new DefaultParameterNameDiscoverer();
        //获取指定的方法，第二个参数可以不传，但是为了防止有重载的现象，还是需要传入参数的类型
        Method method = Class.forName(classType).getMethod(methodName, classes);
        String[] parameterNames = pnd.getParameterNames(method);

        // 将参数名，参数值放入map返回
        HashMap<String, Object> map = new HashMap<>(16);
        for (int k = 0; k < args.length; k++) {
            assert parameterNames != null;
            map.put(parameterNames[k], args[k]);
        }
        return map;
    }
}

```

##### 业务逻辑记录日志

添加日志变成了，只需在方法和类上添加注解就可以搞定，添加日志变得如此简单！！！

```java
/**
 * @author: ***
 * @date: 2019/12/19 10:36
 */
@Service
@LogFunction(moduleName = "日志服务器")
public class SysLogServerServiceImpl implements SysLogServerService {
    private Logger logger = LoggerFactory.getLogger(SysLogServerServiceImpl.class);

    @Autowired
    private MongoTemplate mongoTemplate;

    @Override
    @LogOperate(operateName = "设置日志服务器地址", operateObj = "云盾", remark  = "添加第三方syslog日志服务器地址")
    public ResResult setSyslog(String serverIp, String port) {
        String sb = "bash " + "/usr/src/cloudsec/shell/setSyslog.sh " 
            + " " + serverIp + " " + port;
        String exec = ShellCommandUtil.exec(sb);
        boolean success = exec != null && exec.contains("success");
        if (!success) {
            return ResUtil.error(CodeList.ERR_SPAWN, "执行shell失败!!!");
        }

        // 将ip/port信息存入数据库
        SysLogTpServer sysLogTpServer = new SysLogTpServer();
        sysLogTpServer.setPort(port);
        sysLogTpServer.setIp(serverIp);
        mongoTemplate.remove(SysLogTpServer.class);
        mongoTemplate.insert(sysLogTpServer);

        return ResUtil.error(CodeList.SUCCESS, " 添加第三方syslog日志服务器地址成功! ");
    }
}
```



#### 重构后页面效果

![img](/img/6-cloud/Csec/syslog/1582627120643.png)