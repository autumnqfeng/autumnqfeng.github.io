---
layout: post
title: "正则表达式分组模式"
subtitle: "捕获性分组与非捕获性分组及前瞻后顾(断言)"
author: "Autu"
header-style: text
tags:
  - 前端
  - JavaScript
  - 正则
---

> **javascript正则表达式里分组模式以小括号来()表示分组，例：/([a-z])/**

#### 一、捕获性分组：()

捕获性分组工作模式()会把每个分组里匹配的值保存起来。

比如利用捕获性分组把 hello world 互换成 world hello：

##### ①方法一：通过exec函数

```js
var str = 'hello world';            //首先创建好字符串
var pattern = /([a-z]+)\s([a-z]+)/; //先通过正则匹配这个字符串，用分组模式来获取这两个单词
var arr = pattern.exec(str); // exec方法返回的是一个数组，包含匹配到的字符串以及分组(也称子串)里的值

console.log(arr); //['hello world','hello','world']  
console.log(arr[0]); //'hello world' 匹配到的字符串
console.log(arr[1]); //'hello' 第一个分组([a-z]+)的值
console.log(arr[2]); //'world' 第二个分组([a-z]+)的值

//这时候两个分组的值都得到了,接下来用字符串拼接法实现互换
var n_str = arr[2]+' '+arr[1];
console.log(n_str) //world hello
```

##### ②方法二：通过属性$1-9

```js
var str = 'hello world';            
var pattern = /([a-z]+)\s([a-z]+)/; 
pattern.test(str); //这个地方必须运行正则匹配一次，方式不限，可以是test()、exec()、以及String的正则方式

console.log(RegExp.$1) //'hello' 第一个分组([a-z]+)的值
console.log(RegExp.$2) //'world' 第二个分组([a-z]+)的值

var n_str = RegExp.$2+' '+RegExp.$1;
console.log(n_str) //world hello
```

##### ③方法三：通过String的replace()

```js
var str = 'hello world'; 
var pattern = /([a-z]+)\s([a-z]+)/; 
var n_str = str.replace(pattern,"$2 $1"); //这里的$1、$2与方法二里的RegExp.$1、RegExp.$2作用是相同的。
console.log(n_str) //world hello
```

#### 二、非捕获性分组：(?:)

非捕获性分组工作模式下分组(?:)会作为匹配校验，并出现在匹配结果字符里面，但不作为子匹配返回。

比如利用非捕获性分组获取字符串000aaa111，而且只返回一个值为aaa111的数组：

```js
//先看用捕获性分组匹配会返回什么
var str1 = '000aaa111';             
var pattern = /([a-z]+)(\d+)/; //捕获性分组匹配
var arr = pattern.exec(str1);  
console.log(arr) //['aaa111','aaa','111']   结果子串也获取到了，这并不是我们想要的结果


//非捕获性分组
var str2 = '000aaa111';
var pattern2 = /(?:[a-z]+)(?:\d+)/; //非捕获性分组匹配
var arr2 = pattern2.exec(str2);  
console.log(arr2) //['aaa111']  结果正确
```

#### 三、前瞻：(?=)和(?!)

前瞻分为正向前瞻和反(负)向前瞻，正向前瞻(?=表达式)表示后面要有什么，反向前瞻(?!=表达式)表示后面不能有什么。

前瞻分组会作为匹配校验，但不出现在匹配结果字符里面，而且不作为子匹配返回。

**正向前瞻**匹配一批图片格式：

```js
//正向前瞻，匹配.jpg后缀文件名
var str = '123.jpg,456.gif,abc.jpg';
var partern = /\w+(?=\.jpg)/g; //正向前瞻匹配
console.log(str.match(partern)); //['123', 'abc']   返回结果正确，没有匹配456.gif
```

**反向前瞻**匹配一批字母加数字：

```js
//反向前瞻，匹配3个及以上的a，而且后面不能有000的字符
var str = 'aaa000 aaaa111 aaaaaaa222';
var partern = /a{3,}(?!000)/g; //反向前瞻匹配
console.log(str.match(partern)); //['aaaa', 'aaaaaaa']   返回结果正确，没有匹配aaa000
```

#### 四、后顾：(?<=)和(?<!) —JavaScript不支持

后顾分为正向后顾和反(负)向后顾，正向后顾 **\(** **\?** **\<** **\=** **表达式** **\)**  表示前面要有什么，反向后顾 **\(** **\?** **\<** **\！** **表达式** **\)**  表示前面不能有什么。


```js
//正向后顾
(?<=abc) //前面需要有abc

//反向后顾
(?<!abc) //前面不能有abc
```

**名词解释:**

前瞻 = 先行断言 

(?=) 正向前瞻 = 正向零宽先行断言 

(?!) 反向前瞻 = 负向前瞻 = 负向零宽先行断言

<br>

后顾 = 后发断言 

**\(** **\?** **\<** **\=** **\)**  正向后顾 = 正向零宽后发断言 

**\(** **\?** **\<** **\!** **\)** 反向后顾 = 负向后顾 = 负向零宽后发断言
