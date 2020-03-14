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
    /** 登录用户 */
    private String username;
    /** 登录ip */
    private String loginIp;
    /** 日志类型 */
    private int logTypeId;
    /** 日志级别 */
    private String level;
    /** 事件类型 */
    private String eventType;
    /** 操作详情 */
    private String remark;
    /** 目标方法参数信息 */
    private String targetMethodFields;
    /** 操作结果(success/fail) */
    private String result;
    /** 日志产生时间 */
    private String time;
    /** 日志产生的时间戳 */
    private Long timestamp;
    /** 失败原因 */
    private String failureReasons;

    public Log() {
    }

    public Log(int logTypeId, String eventType, String remark, String time, Long timestamp, String level) {
        this.logTypeId = logTypeId;
        this.eventType = eventType;
        this.remark = remark;
        this.time = time;
        this.timestamp = timestamp;
        this.level = level;
    }

    public Log(int logTypeId, String eventType, String remark, String targetMethodFields, String time, Long timestamp) {
        this.logTypeId = logTypeId;
        this.eventType = eventType;
        this.remark = remark;
        this.targetMethodFields = targetMethodFields;
        this.time = time;
        this.timestamp = timestamp;
    }
    
    // toString和set、get方法略
}
```

##### 创建注解

-   创建作用在方法上的注解：记录详细的日志信息

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
        // 日志类型id
        int logTypeId() default 0;
        // 事件类型
        String eventType() default "";
        // 操作详情
        String remark() default "";
    }
    
    
  ```

  

##### 定义切面类

###### 获取切入点 目标方法中的参数信息

```java
    /**
     * 获取目标方法中的参数信息
     * @param joinPoint 切入点对象
     * @return key: 参数名, value: 参数值
     */
    private HashMap<String, Object> getFieldsMsg(JoinPoint joinPoint) {
        // 获取目标方法参数名
        Signature signature = joinPoint.getSignature();
        MethodSignature methodSignature = (MethodSignature) signature;
        String[] argsName = methodSignature.getParameterNames();
        // 获取目标方法参数
        Object[] args = joinPoint.getArgs();

        HashMap<String, Object> map = new HashMap<>(16);
        for (int i = 0; i < args.length; i++) {
            if (args[i] != null) {
                map.put(argsName[i], args[i]);
            }
        }
        return map;
    }
```

###### 完整切面类代码

- 切入点：所有service包下带LogOperate注解的方法。

- 环绕通知（@Around）：获取方法返回值

- 后置通知（@After）：对存入日志信息的处理

```java
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
    @Autowired
    private SysLogUtil sysLogUtil;

    /** 操作结果 */
    private String result;
    /** 日志级别 */
    private String level;
    /** 失败原因 */
    private String failureReasons;

    @Pointcut("execution(* com.csec.csos.service.*.*(..)) && @annotation(com.csec.csos.annotation.LogOperate)")
    public void pointCut() {}

    // 获取方法返回值
    @Around("pointCut()")
    public Object around(ProceedingJoinPoint joinPoint) throws Throwable {
        // 改方式会执行2次切入点方法
        // if (joinPoint.proceed() instanceof ResResult) {
        //     ResResult resResult = (ResResult) joinPoint.proceed();
        // 次方法只能执行一次，若执行多次则切入点方法也将执行同样次数
        Object proceed = joinPoint.proceed();
        if (proceed instanceof ResResult) {
            ResResult resResult = (ResResult) proceed;
            if (CodeList.SUCCESS == resResult.getCode()) {
                result = "success";
                level = SyslogLevel.INFO;
                failureReasons = "";
            } else {
                result = "fail";
                level = SyslogLevel.ERROR;
                failureReasons = resResult.getMsg();
            }
            return resResult;
        } else {
            return null;
        }
    }

    @After("pointCut()")
    public void after(JoinPoint joinPoint) {
        Log logInfo = getLogAnnotationInfo(joinPoint);
        logInfo.setResult(result);
        logInfo.setLevel(level);
        logInfo.setFailureReasons(failureReasons);
        // 获取用户登录信息
        ConcurrentHashMap<String, String> userInfo = sysLogUtil.getUserInfo();
        logInfo.setUsername(userInfo.get("username"));
        logInfo.setLoginIp(userInfo.get("realIp"));
        mongoTemplate.insert(logInfo);
    }

    /**
     * 获取注解中信息
     * @param joinPoint 切入点对象
     * @return 返回系统操作日志对象
     */
    private Log getLogAnnotationInfo(JoinPoint joinPoint) {
        // 获取日志注解中信息
        MethodSignature signature = (MethodSignature) joinPoint.getSignature();
        Method method = signature.getMethod();
        LogOperate annotationOperate = method.getAnnotation(LogOperate.class);

        // 遍历参数信息
        StringBuilder targetMethodFieldsBuilder = new StringBuilder();
        Map<String, Object> targetMethodFields = this.getFieldsMsg(joinPoint);
        if (targetMethodFields != null) {
            for (Map.Entry<String, Object> param : targetMethodFields.entrySet()) {
                targetMethodFieldsBuilder.append(param.getKey());
                targetMethodFieldsBuilder.append("=");
                targetMethodFieldsBuilder.append(param.getValue());
                targetMethodFieldsBuilder.append("；");
            }
        }
        return new Log(annotationOperate.logTypeId(),
                annotationOperate.eventType(),
                annotationOperate.remark(),
                targetMethodFieldsBuilder.toString(),
                DateUtil.getDateTime(),
                System.currentTimeMillis());
    }

    /**
     * 获取目标方法中的参数信息
     * @param joinPoint 切入点对象
     * @return key: 参数名, value: 参数值
     */
    private HashMap<String, Object> getFieldsMsg(JoinPoint joinPoint) {
        // 获取目标方法参数名
        Signature signature = joinPoint.getSignature();
        MethodSignature methodSignature = (MethodSignature) signature;
        String[] argsName = methodSignature.getParameterNames();
        // 获取目标方法参数
        Object[] args = joinPoint.getArgs();

        HashMap<String, Object> map = new HashMap<>(16);
        for (int i = 0; i < args.length; i++) {
            if (args[i] != null) {
                map.put(argsName[i], args[i]);
            }
        }
        return map;
    }

}
```



##### SysLogUtil中添加公共方法

###### 获取用户登录信息

IP地址在cookie中获取的原因：获取到的ip一直为127.0.0.1，这个ip是前端web代理服务器express的部署地址，因此需要在代理服务器将真实ip放入cookie中传入后端业务代码

- webmain.js中获取真实ip

  ```js
  function getClientIp(req) {
          return req.headers['x-forwarded-for'] ||
          req.connection.remoteAddress ||
          req.socket.remoteAddress ||
          req.connection.socket.remoteAddress;
  }
  ```

- webmain.js中设置cookie（此处为代理拦截路径）

  ```js
  // 登录网站
  app.get('/', function(req, res, next) {
      res.cookie('realIp',getClientIp(req));
      debugger;
      if (!req.cookies.token || req.cookies.token == '0' || req.cookies.token == 'null') {
          // return res.redirect('/login');
      } else {
          return res.redirect('/ce_network/main_page');
      }
      next()
  });
  ```

- 获取用户ip以及用户名信息

  ```java
      /**
       * 获取用户登录信息
       * @return 用户名、密码
       */
      public ConcurrentHashMap<String, String> getUserInfo() {
          ConcurrentHashMap<String, String> userInfoMap = new ConcurrentHashMap<>(16);
  
          //获取request
          ServletRequestAttributes sra = (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
          assert sra != null;
          HttpServletRequest request = sra.getRequest();
          // 获取cookie
          Cookie[] cookies = request.getCookies();
          for (Cookie cookie : cookies) {
              //获取用户名
              if ("token".equals(cookie.getName())) {
                  UserToken token = mongoTemplate.findOne(Query.query(Criteria.where("token").is(cookie.getValue())), UserToken.class);
                  String username = token != null ? token.getUsername() : null;
                  assert username != null;
                  userInfoMap.put("username", username);
              }
              //获取用户真实ip地址
              if ("realIp".equals(cookie.getName())) {
                  String decodeIp = URIDecoder.decodeURIComponent(cookie.getValue());
                  String realIp;
                  if (decodeIp.contains("::ffff:")) {
                      realIp = decodeIp.substring(decodeIp.lastIndexOf(":") + 1);
                  } else if ("::1".equals(decodeIp)) {
                      realIp = "127.0.0.1";
                  } else {
                      realIp = decodeIp;
                  }
                  userInfoMap.put("realIp", realIp);
              }
          }
          return userInfoMap;
      }
  ```

###### 设置系统日志方法

- 非aop方式的增加系统操作日志，弥补aop的不足

  ```java
      /**
       * 设置系统日志
       * @param logTypeId 日志类型id  SyslogType中类型
       * @param eventType 事件类型
       * @param remark 描述信息
       * @param result 日志结果
       * @param failureReasons 失败原因
       * @param level 日志级别
       */
      private void setLog(int logTypeId, String eventType, String remark, String result, String failureReasons, String level) {
          ConcurrentHashMap<String, String> userInfo = getUserInfo();
          Log logInfo = new Log(logTypeId, eventType, remark, DateUtil.getDateTime(), System.currentTimeMillis(), level);
          logInfo.setResult(result);
          logInfo.setFailureReasons(failureReasons);
          logInfo.setUsername(userInfo.get("username"));
          logInfo.setLoginIp(userInfo.get("realIp"));
          mongoTemplate.insert(logInfo);
      }
  ```


###### SysLogUtil完整代码

```java
/**
 * @author: ***
 * @date: 2020/2/27 18:09
 */
@Component
public class SysLogUtil {

    @Autowired
    private MongoTemplate mongoTemplate;

    /**
     * 设置通知级别系统日志
     * @param logTypeId 日志类型id  SyslogType中类型
     * @param eventType 事件类型
     * @param remark 描述信息
     */
    public void setInfoLog(int logTypeId, String eventType, String remark) {
        setLog(logTypeId, eventType, remark, "success", "", SyslogLevel.INFO);
    }

    /**
     * 设置错误级别日志
     * @param logTypeId 日志类型id  SyslogType中类型
     * @param eventType 事件类型
     * @param remark 描述信息
     * @param failureReasons 失败原因
     */
    public void setErrorLog(int logTypeId, String eventType, String remark, String failureReasons) {
        setLog(logTypeId, eventType, remark, "fail", failureReasons, SyslogLevel.ERROR);
    }

    /**
     * 获取用户登录信息
     * @return 用户名、密码
     */
    public ConcurrentHashMap<String, String> getUserInfo() {
        ConcurrentHashMap<String, String> userInfoMap = new ConcurrentHashMap<>(16);

        //获取request
        ServletRequestAttributes sra = (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
        assert sra != null;
        HttpServletRequest request = sra.getRequest();
        // 获取cookie
        Cookie[] cookies = request.getCookies();
        for (Cookie cookie : cookies) {
            //获取用户名
            if ("token".equals(cookie.getName())) {
                UserToken token = mongoTemplate.findOne(Query.query(Criteria.where("token").is(cookie.getValue())), UserToken.class);
                String username = token != null ? token.getUsername() : null;
                assert username != null;
                userInfoMap.put("username", username);
            }
            //获取用户真实ip地址
            if ("realIp".equals(cookie.getName())) {
                String decodeIp = URIDecoder.decodeURIComponent(cookie.getValue());
                String realIp;
                if (decodeIp.contains("::ffff:")) {
                    realIp = decodeIp.substring(decodeIp.lastIndexOf(":") + 1);
                } else if ("::1".equals(decodeIp)) {
                    realIp = "127.0.0.1";
                } else {
                    realIp = decodeIp;
                }
                userInfoMap.put("realIp", realIp);
            }
        }
        return userInfoMap;
    }

    /**
     * 设置系统日志
     * @param logTypeId 日志类型id  SyslogType中类型
     * @param eventType 事件类型
     * @param remark 描述信息
     * @param result 日志结果
     * @param failureReasons 失败原因
     * @param level 日志级别
     */
    private void setLog(int logTypeId, String eventType, String remark, String result, String failureReasons, String level) {
        ConcurrentHashMap<String, String> userInfo = getUserInfo();
        Log logInfo = new Log(logTypeId, eventType, remark, DateUtil.getDateTime(), System.currentTimeMillis(), level);
        logInfo.setResult(result);
        logInfo.setFailureReasons(failureReasons);
        logInfo.setUsername(userInfo.get("username"));
        logInfo.setLoginIp(userInfo.get("realIp"));
        mongoTemplate.insert(logInfo);
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
    @LogOperate(logTypeId = SyslogType.GLOBA_LCONFIG, eventType = "新增", remark  = "添加第三方syslog日志服务器地址")
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

![img](/img/6-cloud/Csec/syslog/1583239987164.png)