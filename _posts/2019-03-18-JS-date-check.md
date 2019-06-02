---
layout: post
title: "JS对日期、时间校验"
subtitle: "兼容IE new Date()"
author: "Autu"
header-style: text
tags:
  - 前端
  - JavaScript
  - 校验
---

### 1. IE浏览器 new Date(); 的坑

```js
var thisDate=new Date("yyyy-MM-dd HH:mm:ss");//此方式IE不支持
var thisDate=new Date("yyyy/MM/dd HH:mm:ss");//此语法支持IE
```
### 2. 比较两个日期的大小。例：2019-3-19

```js
	//比较两个日期大小
	function dateCompare(startDate,endDate){
	   var aStart=startDate.split('-'); //转成成数组，分别为年，月，日，下同
	   var aEnd=endDate.split('-');
	   var startDateTemp = aStart[0]+"/" + aStart[1]+ "/" + aStart[2];
	   var endDateTemp = aEnd[0] + "/" + aEnd[1] + "/" + aEnd[2];
	   if (startDateTemp > endDateTemp) 
	    return true;
	   else
	    return false;
	}
```
### 3. 计算两个日期之间的天数

```js
	function dateDiff(date1, date2){     
		var type1 = typeof date1, type2 = typeof date2;     
		if(type1 == 'string')     
			date1 = stringToTime(date1);     
		else if(date1.getTime)     
			date1 = date1.getTime();     
		if(type2 == 'string')     
			date2 = stringToTime(date2);     
		else if(date2.getTime)     
			date2 = date2.getTime(); 
		return (date2 - date1) / 1000 / 60 / 60 / 24;//除1000是毫秒，不加是秒 
	} 
	//字符串转成Time(dateDiff)所需方法 
	function stringToTime(string){     
		var f = string.split(' ', 2);     
		var d = (f[0] ? f[0] : '').split('-', 3);     
		var t = (f[1] ? f[1] : '').split(':', 3);     
		return (new Date(     
		parseInt(d[0], 10) || null,     
		(parseInt(d[1], 10) || 1)-1,     
		parseInt(d[2], 10) || null,     
		parseInt(t[0], 10) || null,    
		parseInt(t[1], 10) || null,     
		parseInt(t[2], 10) || null)).getTime(); 
	}
```

### 4. 校验2019-04-01类型的日期，闰年、平年均可校验
```js
/**
 * 日期格式校验
 * @param date
 * @returns {Boolean} 校验错误返回false
 */
function checkData(date){
	var reg = /^((?!0000)[0-9]{4}-((0[1-9]|1[0-2])-(0[1-9]|1[0-9]|2[0-8])|(0[13-9]|1[0-2])-(29|30)|(0[13578]|1[02])-31)|([0-9]{2}(0[48]|[2468][048]|[13579][26])|(0[48]|[2468][048]|[13579][26])00)-02-29)$/;
	if (!reg.test(date)) {
		alert("date format illegal,The correct format is: yyyy-mm-dd !");
		return false;
	}
}
```
