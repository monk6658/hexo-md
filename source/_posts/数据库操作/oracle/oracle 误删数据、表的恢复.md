---
title: oracle 误删数据、表的恢复 #文章页面上的显示名称，一般是中文
date: 2020-10-21 14:16:33 #文章生成时间，一般不改，当然也可以任意修改
categories: oracle #分类
tags: [运维,tag2,tag3] #文章标签，可空，多标签请用格式，注意:后面有个空格
description: 附加一段文章摘要，字数最好在140字以内，会出现在meta的description里面
---

<font size="4" color="#dd0000">注意：鉴于oracle的特殊性，表名基本为大写（注意大小写问题）。</font>

## 一、delete 误删数据问题

### 1、根据v$sqlarea进行恢复，把表恢复到删除时间点
1. 查询sql执行时间

```
select SQL_TEXT,LAST_ACTIVE_TIME from v$sqlarea 
where LAST_ACTIVE_TIME > to_date('2020-07-09 9:30:27','yyyy-mm-dd hh24:mi:ss')
and SQL_TEXT like '%表名%';
```

2. 根据时间进行恢复

```oracle
INSERT INTO 表名
select * from 表名 AS OF TIMESTAMP to_timestamp('2020-07-09 9:32:27','yyyy-mm-dd hh24:mi:ss');
```
### 2、根据闪回把表恢复到某个时间点

```oracle
-- 直接闪回某张表到某个时间点
flashback table 表名 to timestamp to_date('2020-07-09 09:49:20','yyyy-mm-dd hh24:mi:ss');
-- 出现异常则先授权，开启行移动功能 
alter table 表名 enable row movement;
```
### 3、通过scn进行恢复（根据scn可把整个库恢复到某个时间点）
```oracle
-- 查询某个时间点的scn
select timestamp_to_scn(to_timestamp('2020-07-09 09:49:20','yyyy-mm-dd hh24:mi:ss')) from dual;

-- 通过闪回把某张表恢复到某个时间点的scn
flashback table 表名 to scn 161664467;
```
## 二、drop 误删表的恢复方法
### 1、通过回收站恢复
```oracle
-- 查询被删除的表
select * from recyclebin where original_name like '%表名%'  order by droptime desc;
-- 恢复被删除的表(如果该表重新被创建，需要恢复到新的表)（如果该表存在多次删除历史，只会恢复最后版本）
flashback table '需要恢复的表名' to before drop [rename to 新表名]；

-- 通过唯一标识(如果该表重新被创建，需要恢复到新的表)（根据recyclebin 的object_name 作为唯一标识恢复指定时间版本）
flashback table "BIN$BuhrIJqTR7qMITOgqKK8qg==$0" to before drop rename to 新表名;
```