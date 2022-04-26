---
title: mysql8
author: ygdxd
type: 原创
date: 2021-07-15 22:10:51
tags: mysql
categories: mysql
---

mysql 8 新特性
===================================

mysql分库分表并不是一定从性能的角度上需求，大部分场景是为了更好的管理数据

1. 数据量大时需要对数据进行新加字段进行加以归类标记，DDL会锁表导致长时间堵塞。(mysql 8 特性会大幅度缩短这个时间，同时mysql 8增加了备份锁，减少备份的时间)
2. 当一张表或者一个库出现问题时不会导致整个应用不能访问db.
3. 灰度？数据库性能。

WHERE 语句内使用函数走索引
-------------

select * from x where YEAR(date) = 2021


降序索引 
------------

mysql索引以前一直都是顺序，现在可以按降序存储。

index idx_c1_c2(c1,c2 desc)
在索引反向扫描是 explain 语句extra中会显示Backward index scan


GROUP BY 语句不在隐式排序
------------
现在select group by 语句不会按照group by 字段进行隐式排序
mysql 5x 中explain 语句的extra 中有Using filesort

在高并发的情况下性能提升
-----------
在高并发的情况下，只读和更新操作性能提升

默认字符集为utf8mb4
-----------
修改默认字符集，同时查询性能增加


增加SKIP LOCKED 和NOWAIT
----------
select * from x where id = 1 for update skip locked
select * from x where id = 1 for update skip nowait

前者查询加锁的记录被其他线程持有锁时会直接返回空。
后者查询加锁的记录被其他线程持有锁时会直接报错。 
在高并发下比如抢红包等场景下使用。

文档数据库
-----------
mysql 支持nosql的crud.同时提供了JSON函数

新增统计函数和GIS
-----------

窗口函数。(不常用)
GIS 地理信息(不常用)

DDL 原子性
----------
数据字典集中存储，增强crash safe能力。

MGR 增强
---------
MGR 集群的健壮和稳定性加强。

自增主键id存储 在删除一条数据后又新增一条数据
--------
在mysql8之前,自增主键的最大值存储在内存中。mysql8将自增id在事务结束checkpoint时放到了redolog中。所以mysql8之前id会持续增大而8会使用之前的。

![mysql] (/images/mysql8_id_inc.jpg)
