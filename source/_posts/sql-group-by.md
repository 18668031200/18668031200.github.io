---
title: GROUP BY分组查询与SQL执行顺序
author: ygdxd
type: 转载
date: 2021-08-08 21:42:09
tags: mysql
category: mysql
---


转自：http://blog.163.com/shexinyang@126/blog/static/1367393122013526113822666/ 


1.SELECT语句使用GROUP BY的一些规定
----------------

+ GROUP BY子句可以包含任意数目的列。也就是说可以在组里再分组，为数据分组提供更细致的控制。
+ 如果在GROUP BY子句中指定多个分组，数据将在最后指定的分组上汇总。
+ GROUP BY子句中列出的每个列都必须是检索列或有效的表达式（但不能是聚集函数）。如果在SELECT中使用了表达式，则必须在GROUP BY子句中指定相同的表达式。不能使用别名。
+ 除了聚集计算语句外，GROUP BY子句中的每一列都必须在SELECT语句给出。
+ 如果分组列中有NULL值，则NULL将作为一个分组返回。如果有多行NULL值，它们将分为一组。
+ GROUP BY子句必须在WHERE子句之后，ORDER BY之前。

2.过滤分组
---------------
对分组过于采用HAVING子句。HAVING子句支持所有WHERE的操作。HAVING与WHERE的区别在于WHERE是过滤行的，而HAVING是用来过滤分组。

另一种理解WHERE与HAVING的区别的方法是，WHERE在分组之前过滤，而HAVING在分组之后以每组为单位过滤。

3.分组与排序
--------------
一般在使用GROUP BY子句时，也应该使用ORDER BY子句。这是保证数据正确排序的唯一方法。

SQL SELECT语句的执行顺序：

from子句组装来自不同数据源的数据(若有join，则先执行on后条件，再连接数据源)；
where子句基于指定的条件对记录行进行筛选；
group by子句将数据划分为多个分组；
使用聚集函数进行计算；
使用having子句筛选分组；
计算所有的表达式；
使用order by对结果集进行排序；
select 集合输出。
