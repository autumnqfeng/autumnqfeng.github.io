---
layout:     post
title: "js监听鼠标移动，超时返回登录页"
date:       2020-02-23 13:36:00
author:     "Autu"
header-img: "img/post-bg-js-module.jpg"
header-mask: 0.3
catalog:    true
tags:
  - 6_Cloud
  - CSEC
  - js
  - 前端
---

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

超时时间在页面可动态设置与获取，因此超时时间存在数据库中，当检测到鼠标不移动时间超过超时时间时，清除cookie，并返回登录页。

为了避免频繁获取数据库超时时间就需要将超时时间放入cookie中，定时获取cookie中的超时时间，当发现cookie中timeOut消失则从后端获取，再重新设置timeOut

代码如下：

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

