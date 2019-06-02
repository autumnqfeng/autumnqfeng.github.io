---
layout: post
title: "数据库表横纵向转换"
subtitle: "经典案例"
author: "Autu"
header-style: text
tags:
  - 数据库
  - SQL
---

### 纵向转横向

name | subject | result
---|---|---
张三 | 语文 | 73
张三 | 数学 | 83
张三 | 物理 | 93
李四 | 语文 | 74
李四 | 数学 | 84
李四 | 物理 | 94

转换成

姓名 | 语文 | 数学 | 物理
---|---|---|---
李四 | 74 | 84 | 94
张三 | 73 | 83 | 93

建表语句

```sql
create TABLE TABLE1 (Name varchar(50),Subject varchar(50),Result int) 
INSERT INTO TABLE1 VALUES('张三','语文','73')
INSERT INTO TABLE1 VALUES('张三','数学','83')
INSERT INTO TABLE1 VALUES('张三','物理','93')
INSERT INTO TABLE1 VALUES('李四','语文','74')
INSERT INTO TABLE1 VALUES('李四','数学','84')
INSERT INTO TABLE1 VALUES('李四','物理','94')
```

转换语句

```sql
SELECT * FROM TABLE1

SELECT 
    name AS 姓名,
    SUM(CASE WHEN subject='语文'then Result ELSE 0 end) AS '语文',
    SUM(CASE WHEN subject='数学'then Result ELSE 0 end) AS '数学',
    SUM(CASE WHEN subject='物理'then Result ELSE 0 end) AS '物理'
into #table1  --下面转换用
FROM dbo.TABLE1
GROUP BY Name
```

--说明： 

--sum 是求和 把语文的result的结果加起来<br>
--sum(case when subject='语文' then result else 0 end) AS '语文',<br>
--case是说当subject='语文' 的时候 就对result求和 如果不等于语文 则把result设为0

--等价于

```sql
SELECT 
    name AS 姓名,
    SUM(CASE WHEN subject='语文'then Result end) AS '语文',
    SUM(CASE WHEN subject='数学'then Result end) AS '数学',
    SUM(CASE WHEN subject='物理'then Result end) AS '物理'
FROM dbo.TABLE1
GROUP BY Name
```

--等价于

--max是求出 数学的最大的result求出来

```sql
SELECT 
    name AS 姓名,
    max(CASE WHEN subject='语文'then Result end) AS '语文',
    max(CASE WHEN subject='数学'then Result end) AS '数学',
    max(CASE WHEN subject='物理'then Result end) AS '物理'
FROM dbo.TABLE1
GROUP BY Name
```

### 横向转纵向

姓名 | 语文 | 数学 | 物理
---|---|---|---
李四 | 74 | 84 | 94
张三 | 73 | 83 | 93

转换成

name | subject | result
---|---|---
张三 | 语文 | 73
张三 | 数学 | 83
张三 | 物理 | 93
李四 | 语文 | 74
李四 | 数学 | 84
李四 | 物理 | 94

--语句

```sql
SELECT * FROM #table1

select 姓名 as Name,'语文' as Subject,语文 as Result from #table1 union 
select 姓名 as Name,'数学' as Subject,数学 as Result from #table1 union 
select 姓名 as Name,'物理' as Subject,物理 as Result from #table1 
order by 姓名 desc
```

--说明

--UNION 指令的目的是将两个 SQL 语句的结果合并起来<br>
--使用union主要注意两表字段数量相等，相应的字段类型应该也应该一致。
