---
layout:     post
title: "js监听鼠标移动，超时返回登录页"
date:       2020-02-23 13:36:00
author:     "Autu"
header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
  - 6_Cloud
  - CSEC
  - js
  - 前端
---

#### 功能描述与当前现状

- 系统超时时间功能，该功能可  **动态 ** 设置超时时间，超时退出登录
- 在之前功能中，超时只是设置token过期，后端删除token，前端并未删除
- 因此超时后若无点击事件产生，系统 **不会退出** 



#### 解决办法与遇到的问题

##### 问题1 ：超时未退出

- 在前端加入鼠标不移动事件
- 监听到鼠标不移动，超时后退出登录

##### 问题2 ：频繁请求后端数据库

- 由于超时时间存在数据库，因此需要频繁访问数据库
- 加入timeOut的cookie，将超时时间存于cookie，定时从cookie中获取超时时间

##### 问题3 ：timeOut消失后超时退出失效

- timeOut可能会被意外删除，或初始化失败
- 定时获取timeOut时，若在cookie中未发现timeOut则从后端获取一次，放入cookie



#### js中setCookie、getCookie工具代码

```js
    //cname 名字
    //cvalue 值
    //exdays 时间            0.01大概25分钟
    function setCookie(cname, cvalue, exdays) {
        var d = new Date();
        d.setTime(d.getTime() + (exdays * 24 * 60 * 60 * 1000));
        var expires = "expires=" + d.toUTCString();
        // expires + "; path=/" 这个很重要代表在哪个层级下可以访问cookie
        document.cookie = cname + "=" + cvalue + "; " + expires + "; path=/";

    }

    //获取cookie
    function getCookie(cname) {
        var name = cname + "=";
        var ca = document.cookie.split(';');
        for(var i = 0; i < ca.length; i++) {
            var c = ca[i];
            while(c.charAt(0) == ' ') c = c.substring(1);
            if(c.indexOf(name) != -1) return c.substring(name.length, c.length);
        }
        return "";
    }

    //删除 cookie
    function clearCookie(name) {
        setCookie(name, "", -1);
    }
```



#### 登录超时时间获取

```js
    let timeOut;
    // 每隔一分钟获取一次cookie中的超时时间
    window.onload = function () {
        timeOutOneMinute();
    };
    function timeOutOneMinute() {
        // 设置超时时间，并回调当前函数 
        setTimeout( timeOutOneMinute, 1000 * 60 );
        timeOut = parseInt(getCookie("timeOut"));
        // 若未检测到timeOut的cookie则从后端获取
        if (!getCookie("timeOut")) {
            jutil.ajax({
                url: '/cloudenforce/api/v4/GetWebTimeout',
                type: "get",
                success: function(data) {
                    if(data && data.code == 0){
                        $.cookie('timeOut', data.data, { path: '/' });
                    }
                },
                error:function(){
                    jutil.msg("获取超时时间失败",'error');
                }
            });
        }
        // 若存在token则timeOut不变，否则将其置零
        timeOut = getCookie("token") ? timeOut : 0;
    }
```



#### 鼠标不移动事件

```js
    // 鼠标不移动事件
    document.onmousemove = function() {
        window.lastMove = new Date().getTime();
    };
    window.lastMove = new Date().getTime();
	// 此函数作用：定时3秒执行function
    window.setInterval(function() {
        var now = new Date().getTime();
        // timeOue单位为分钟
        if(now - lastMove > timeOut * 1000 * 60) {
            clearCookie('token');
            window.location.href = '/login';
        }
    }, 3000);
```



#### 完整代码：

```js
    //监听鼠标，鼠标没有移动超过系统超时时间，清除cookie。
    let timeOut;
    // 每隔一分钟获取一次cookie中的超时时间
    window.onload = function () {
        timeOutOneMinute();
    };
    function timeOutOneMinute() {
        setTimeout( timeOutOneMinute, 1000 * 60 );
        timeOut = parseInt(getCookie("timeOut"));
        if (!getCookie("timeOut")) {
            jutil.ajax({
                url: '/cloudenforce/api/v4/GetWebTimeout',
                type: "get",
                success: function(data) {
                    if(data && data.code == 0){
                        $.cookie('timeOut', data.data, { path: '/' });
                    }
                },
                error:function(){
                    jutil.msg("获取超时时间失败",'error');
                }
            });
        }
        timeOut = getCookie("token") ? timeOut : 0;
    }

    // 鼠标不移动事件
    document.onmousemove = function() {
        window.lastMove = new Date().getTime();
    };
    window.lastMove = new Date().getTime();
    window.setInterval(function() {
        var now = new Date().getTime();
        if(now - lastMove > timeOut * 1000 * 60) {
            clearCookie('token');
            window.location.href = '/login';
        }
    }, 0);

    //删除 cookie
    function clearCookie(name) {
        setCookie(name, "", -1);
    }
    //获取cookie
    function getCookie(cname) {
        var name = cname + "=";
        var ca = document.cookie.split(';');
        for(var i = 0; i < ca.length; i++) {
            var c = ca[i];
            while(c.charAt(0) == ' ') c = c.substring(1);
            if(c.indexOf(name) != -1) return c.substring(name.length, c.length);
        }
        return "";
    }

    //cname 名字
    //cvalue 值
    //exdays 时间            0.01大概25分钟
    function setCookie(cname, cvalue, exdays) {
        var d = new Date();
        d.setTime(d.getTime() + (exdays * 24 * 60 * 60 * 1000));
        var expires = "expires=" + d.toUTCString();
        document.cookie = cname + "=" + cvalue + "; " + expires + "; path=/";

    }
```

