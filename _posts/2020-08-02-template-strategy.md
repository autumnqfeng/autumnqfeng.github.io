---
layout:     post
title: "模板模式&策略模式 - 多元化登录"
subtitle: "你还在写if else吗"
date:       2020-08-02 12:13:00
author:     "Autu"
header-img: "img/post-bg-web.jpg"
header-mask: 0.3
catalog:    true
tags:
  - 设计模式
  - 实战
---

## 模板模式&策略模式 - 多元化登录

### 一个问题引发的思考

在登录场景中，除了有用户名、密码登录之外

还有常见的手机号验证登录、邮箱验证登录、QQ验证、微信验证等多种登录方式。

那当我们拿到这样的需求一般会怎样去实现呢？

### 传统实现

#### `dto`

```java
@Data
public class AuthLoginDto {
    private String username;
    private String password;

    private String phone;
    private String code;

    private String openId;

    @NotNull(message = "登录类型不能为空")
    private Integer loginType;
    
    public void validData(BindingResult bindingResult){
        if(bindingResult.hasErrors()){
            StringBuilder stringBuilder=new StringBuilder();
            for(ObjectError oe:bindingResult.getAllErrors()){
                stringBuilder.append(oe.getDefaultMessage()+"\n");
            }
            throw new ValidException(stringBuilder.toString());
        }
    }
}
```

#### controller

```java
@PostMapping("/login")
public R loginAuth(@RequestBody @Validated AuthLoginDto authLoginDto, BindingResult bindingResult) {
    authLoginDto.validData(bindingResult);
    // 登录逻辑
    Integer loginType = authLoginDto.getLoginType()；
}

private R doLogin(Integer loginType) {
    switch(loginType){
        case 1:
            // 用户名密码登录
        case 2:
            // 手机验证码登录
        case 3:
            // 手机密码登录
        default:
            // 暂不支持该种登录类型
    }
}
```



我们看到这种实现方式，可以快速实现功能。而且增加登录方式只需增加case即可，业务逻辑可以快速实现。

当增加需求时，我们需要修改原来的类，原来的方法，而且switch的分支会变得非常多。有代码洁癖的人看到这样代码就想如何去修改这样的代码，优化这样的代码。

那么我们应该怎样去优化呢？

设计模式可以帮你解决，我们先了解下策略模式和模板模式，知道了他是什么才能运用自如。



### 策略模式

策略模式是一种行为模式，也是替代大量 `if else` 的利器。他能帮你解决的场景，一般是具有同类可替代的行为逻辑算法场景。

比如：

* 不同类型的交易方式（信用卡、支付宝、微信）
* 不同的登录方式（用户名密码、手机号、邮箱）
* 生成唯一的ID策略（UUID、自增、雪花算法、Leaf算法）

以上场景都可以使用策略模式包装，提供给外部使用。

### 模板模式

模板模式的核心设计思路是通过在抽象类中定义抽象方法的执行顺序，并将抽象方法设定为只有子类实现，但不设计独立访问的方法。简单来说就是把你安排的明明白白的。

### 重构

#### 代码结构

![1596351582728](http://123.57.45.66/images/autu_blog/design_mode/2020-08-02-template-strategy/1596351582728.png)

#### 定义枚举 `LoginTypeEnum`

```java
public enum  LoginTypeEnum {
    NORMAL(0, "帐号密码登录"),
    PHONE_PWD(1, "手机号与密码登录"),
    PHONE_CODE(2, "手机验证码登录"),
    WECHAT(3, "微信授权登录");

    private int code;
    private String memo;

    LoginTypeEnum(int code, String memo) {
        this.code = code;
        this.memo = memo;
    }

    // set get方法略
    ...
}
```

#### 登录接口 `Login`

```java
public interface Login {

    R doLogin(AuthLoginDto authLoginDto) throws BizException;
}
```

#### 登录抽象类 `AbstractLogin`

```java
@Slf4j
public abstract class AbstractLogin implements Login{
	// 类初始化时，将子类对象存储在map中，key：登录类型
    public static ConcurrentHashMap<Integer, AbstractLogin> loginMap = new ConcurrentHashMap<>();

    @PostConstruct
    public void init(){
        loginMap.put(getLoginType(),this);
    }

    @Override
    public R doLogin(AuthLoginDto authLoginDto) throws BizException {
        log.info("begin AbstractLogin.doLogin:"+authLoginDto);
        // 第一步完成参数验证
        validate(authLoginDto);
        // 登录校验
        TbMember member = doProcessor(authLoginDto);
        // jwt生成token
        Map<String,Object> payLoad = new HashMap<>();
        payLoad.put("uid", member.getId());
        payLoad.put("exp", DateTime.now().plusHours(1).toDate().getTime()/1000);
        String token= JwtGeneratorUtil.generatorToken(payLoad);
        return new R.Builder().setData(token).buildOk();
    }

    /**
     * 在子类中去声明自己的登录类型
     * @return
     */
    protected abstract int getLoginType();

    /**
     * 通过子类去完成验证
     * @param authLoginDto
     */
    protected abstract void validate(AuthLoginDto authLoginDto);

    /**
     * 登录校验
     * @param authLoginDto
     */
    protected abstract TbMember doProcessor(AuthLoginDto authLoginDto);

}
```

#### 实现类 - 用户名密码登录

```java
@Slf4j
@Service
public class NormalLoginProcessor extends AbstractLogin{

    @Autowired
    TbMemberMapper tbMemberMapper;

    @Override
    public int getLoginType() {
        return LoginTypeEnum.NORMAL.getCode();
    }

    @Override
    public void validate(AuthLoginDto authLoginDto) {
        if(StringUtils.isBlank(authLoginDto.getUsername()) || StringUtils.isBlank(authLoginDto.getPassword())){
            throw new ValidException("帐号或者密码不能为空");
        }
    }

    @Override
    public TbMember doProcessor(AuthLoginDto authLoginDto) {
        log.info("begin NormalLoginProcessor.doProcessor:" + authLoginDto);
        TbMemberExample tbMemberExample = new TbMemberExample();
        tbMemberExample.createCriteria()
            .andStateEqualTo(1)
            .andUsernameEqualTo(authLoginDto.getUsername());
        List<TbMember> members=tbMemberMapper.selectByExample(tbMemberExample);
        // 数据库验证
        if(members == null || members.size() == 0){
            throw new BizException("用户名或者密码错误");
        }
        // 密码验证
        if(!DigestUtils.md5DigestAsHex(authLoginDto.getPassword().getBytes())
           .equals(members.get(0).getPassword())){
            throw new BizException("用户名或者密码错误");
        }
        return members.get(0);
    }
}
```

#### controller层登录接口

```java
@RestController
public class LoginController {

    @PostMapping("/login")
    public R loginAuth(@RequestBody @Validated AuthLoginDto authLoginDto, BindingResult bindingResult){
        authLoginDto.validData(bindingResult);
        // 登录逻辑, 根据登录类型找到对应的实现类（寻找模板）
        Login login= AbstractLogin.loginMap.get(authLoginDto.getLoginType());
        if(login==null){
            throw new BizException("暂不支持该种登录类型");
        }
        // 执行登录逻辑
        return login.doLogin(authLoginDto);
    }
}
```

### 总结

策略模式：

* 在login方法中通过不同登录类型，交给指定的类执行登录逻辑。（也就是找模板）

* 在 `AbstractLogin` 类加载完成后（ `init()` ），将所有模板对象存储在 `new ConcurrentHashMap<>();` 中

模板模式：

* 首先定义了一个 `Login` 接口，其中定义了登录相关抽象方法

* 在抽象类 `AbstractLogin` 中实现登录相关方法，供外部调用，这个类是**模板模式的灵魂**
* 登录方法分为三步：1、前端登录数据校验，2、`doProcessor` 登录逻辑，3、生成token
* 参数检验、登录均定义为抽象方法，仅供子类访问（ `protected` ）