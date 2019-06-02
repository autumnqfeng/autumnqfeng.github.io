---
layout: post
title: "SQL中 EXCEPT、INTERSECT用法"
subtitle: "差集、交集、并集"
author: "Autu"
header-style: text
tags:
  - 数据库
  - SQL
---

> 
<code>EXCEPT </code>返回两个结果集的差（即从左查询中返回右查询没有找到的所有非重复值）。<br><br>
<code>INTERSECT </code>返回 两个结果集的交集（即两个查询都返回的所有非重复值）。<br><br>
<code>UNION </code>返回两个结果集的并集。

### 语法：

{ （< SQL－查询语句1>） } 

{ EXCEPT&nbsp;/&nbsp;INTERSECT } 

{ （< SQL－查询语句2> ）}

### 限制条件

1. 所有查询中的<code>列数和列的顺序必须相同</code>。
2. 比较的两个查询结果集中的<code>列数据类型</code>可以不同但必须兼容。
3. 比较的两个查询结果集中不能包含不可比较的数据类型（xml、text、ntext、image 或非二进制 CLR 用户定义类型）的列。
4. 返回的结果集的列名与操作数左侧的查询返回的列名相同。<code>ORDER BY</code> 子句中的列名或别名必须引用左侧查询返回的列名。
5. 不能与 <code>COMPUTE </code>和 <code>COMPUTE BY </code>子句一起使用。
6. 通过比较行来确定非重复值时，两个 NULL 值被视为相等。（EXCEPT 或 INTERSECT 返回的结果集中的任何列的为空性与操作数左侧的查询返回的对应列的为空性相同。）

### 与表达式中的其他运算符一起使用时的执行顺序

1. 括号中的表达式
2. INTERSECT 操作数
3. 基于在表达式中的位置从左到右求值的 EXCEPT 和 UNION

如果 EXCEPT 或 INTERSECT 用于比较两个以上的查询集，则数据类型转换是通过一次比较两个查询来确定的，并遵循前面提到的表达式求值规则。

### 举例：

tableA | tableB
---|---
NULL | NULL
NULL | 2
1 | 3
1 | 4
2 | 5
3 | 5
4 |
5 |

**Ａ：**
```sql
（SELECT * FROM TableA） EXCEPT （SELECT * FROM TableB）
```
结果：

    １
    (1 row(s) affected)

**Ｂ：**


```sql
SELECT * FROM TableA INTERSECT SELECT * FROM TableB
```
结果：

    ２
    ３
    ４
    ５
    (４ row(s) affected)

