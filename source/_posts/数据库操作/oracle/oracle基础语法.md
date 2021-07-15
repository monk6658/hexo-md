---
title: oracle基础语法
date: 2020-12-17 #文章生成时间，一般不改，当然也可以任意修改
categories: oracle #分类
tags: [sql,oracle,函数] #文章标签，可空，多标签请用格式，注意:后面有个空格
description: oracle基础的语法及函数使用
---

# 1. oracle

## 1. oracle基础语法

### 1.decode

decode (条件，值1，返回值1，值2，返回值2...，缺省值)

### 2.trunc

#### （1）截断数字

TRUNC（n1,n2），n1表示被截断的数字，n2表示要截断到那一位。n2可以是负数，表示截断小数点前。注意，TRUNC截断不是四舍五入。

```sql
select trunc(123.458,0) from dual; --123
select trunc(123.458,1) from dual; --123.4
select trunc(123.458,-1) from dual; --120
```

#### （2）截断日期

```sql
select trunc(sysdate, 'mm') from dual; --2013-01-01 返回当月第一天.
```

| 值       | 含义                     |
| -------- | ------------------------ |
| mm       | 当月第一天               |
| yy、yyyy | 返回当年第一天           |
| dd       | 当前年月日               |
| d        | 当前星期的第一天(星期天) |
| hh       | 当前时间，精确到小时     |
| mi       | 当前时间，精确到分       |

### 3.add_months

add_months(times,num)：times是时间格式的日期，num为之后几个月，可为负数

### 4. extract

用法：从一个date或者interval类型中截取到特定的部分

语法：

extract ({ year | month | day | hour | minute | second | 某一时区 }
from { date类型值 | interval类型值} )

```sql
select extract (year from sysdate) year from dual
```

### 5. listagg

用法：多行合并成一行

语法：listagg('字段','分隔符') within group(order by 3)

### 6. pivot

用法：多行转多列

语法：

```sql
pivot (聚合函数(列名) [as 别名1] for 列名 in (列值 as 别名2,...));
```

注意：如果使用[as 别名1]，字段名为 **别名1_别名2**。

## 2. oracle 创建表空间及用户

```sql

-- 查看表空间
select * from dba_data_files where tablespace_name = 'KQ_JK'

delete from dba_data_files where tablespace_name = 'KQ_JK';

-- 1. 创建临时表空间
CREATE TEMPORARY TABLESPACE KQ_JK_TEMP
TEMPFILE '/data/oradata/testdb/KQ_JK_TEMP02.dbf'
SIZE 32M
AUTOEXTEND ON
NEXT 32M MAXSIZE UNLIMITED
EXTENT MANAGEMENT LOCAL;
         
         
-- 2. 表空间
CREATE TABLESPACE KQ_JK
LOGGING
DATAFILE '/data/oradata/testdb/KQ_JK01.dbf'
SIZE 32M
AUTOEXTEND ON
NEXT 32M MAXSIZE UNLIMITED
EXTENT MANAGEMENT LOCAL; 

-- 3. 在表空间下创建角色
CREATE USER KQ_JK IDENTIFIED BY oracle
ACCOUNT UNLOCK
DEFAULT TABLESPACE KQ_JK
TEMPORARY TABLESPACE KQ_JK_TEMP;
-- （如果没有创建临时表空间，则不需要这句话）

-- 4. 授权
GRANT CONNECT,RESOURCE TO KQ_JK;  --表示把 connect,resource权限授予tbb用户
GRANT DBA TO KQ_JK;  --表示把 dba权限授予给tbb用户（可选）

```

## 3. 索引未命中出现条件

1. update语句时，varchar 类型字段，使用numer赋值

错误示例: update xxx set str = 3;

正确示例: update xxx set str = '3';

