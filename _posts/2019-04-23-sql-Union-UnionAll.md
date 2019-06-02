---
layout: post
title: "SQL操作符UNION和UNION ALL"
subtitle: "合并两个或多个 SELECT 语句的结果集"
author: "Autu"
header-style: text
tags:
  - 数据库
  - SQL
---

UNION 、UNION ALL操作符用于<code>合并两个或多个 SELECT 语句的结果集</code>。从这个角度来看，它们跟 JOIN 有些类似，都可以从多个表中获取信息。<br><br>
**注意的是：**<br>
- UNION 、UNION ALL内部的 SELECT 语句必须拥有相同数量的列
- 列也必须拥有相似的数据类型
- 同时，每条 SELECT语句中的列的顺序必须相同。

<br>
下面是设计的两张表：<br><br>
<code>STUDENTONE_MESSAGE表</code>

ID | STUDENT_NAME | COURSE
---|---|---
1003 | 小C | Java
1001 | 小A | C++
1002 | 小B | r

<code>STUDENTTWO_MESSAGE表</code>

ID | STUDENT_NAME | COURSE
---|---|---
1002 | 小小B | Java
1001 | 小A | C++

**1.采用UNION操作符：**


```sql
select id,student_name,course FROM studentone_message 
UNION 
select id,student_name,course FROM studenttwo_message
```

查询结果：

ID | STUDENT_NAME | COURSE
---|---|---
1003 | 小C | Java
1001 | 小A | C++
1002 | 小B | r
1002 | 小小B | Java

**2.采用UNION ALL操作符：**


```sql
select id,student_name,course FROM studentone_message
UNION ALL
select id,student_name,course FROM studenttwo_message
```

查询结果：

ID | STUDENT_NAME | COURSE
---|---|---
1003 | 小C | Java
1001 | 小A | C++
1002 | 小B | r
1002 | 小小B | Java
1001 | 小A | C++

**总结：**<br>
1.通过以上信息可以看出，<code>UNION</code> 操作符合并的结果集，<code>不允许重复值</code>；<code>UNION ALL允许有重复值</code>。<br>
2.但是UNION 会将各查询子集的记录做比较，所以相对于UNION ALL来说 ，<code>UNION 速度会慢上许多</code>。一般来说，在确保查询数值不会重复的前提下，要用UNION ALL。
