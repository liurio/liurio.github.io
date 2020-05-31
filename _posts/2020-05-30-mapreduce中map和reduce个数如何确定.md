---
logout: post
title: mapreduce中map和reduce个数如何确定
tags: [hive,all]
---

## 控制map数

通常情况下，作业会通过input的目录产生一个或多个map任务。主要决定因素有：input的文件总个数、input的文件大小、集群设置的文件块大小(128M)。

### 减少map数

如果一个任务有很多**小文件（远远小于128M）**，则每个小文件也会被当做一个块，用一个map任务来完成。而一个map任务的启动和初始化的时间远远大于逻辑处理时间，就会造成资源浪费。

- 问题

例如一个sql任务如下：

```sql
Select count(1) from popt_tbaccountcopy_mes where pt = ‘2012-07-04’;
```

该任务的目录`popt_tbaccountcopy_mes/pt=2012-07-04`中共有194个文件，其中很多远远小于`128M`，总大小是9G，正常执行会用194个任务。

- 解决

首先要在map执行之前合并小文件，减少map数：

```sql
set mapred.max.split.size = 100000000;	--1
set mapred.min.split.size.per.node = 100000000;	--2
set mapred.min.split.size.per.rack = 100000000;	--3
set hive.input.format = org.apache.hadoop.hive.ql.io.CombineHiveInputFormat;--4
```

其中100000000表示100M，第四个设置是执行前进行小文件的合并，1、2、3三个参数确定合并文件块的大小，`>128M`：按照128分割；`100M<size<128M`：按照100M分割；`<100M`：进行合并。最终分成74个块。

### 增加map数

当input文件都很大时，任务逻辑复杂，map执行非常慢的时候，可以考虑增加Map数，来使得每个map处理的数据量减少，从而提高任务的执行效率。

- 问题

例如一个sql任务如下：

``` sql
select d,count(1) from a group by d
```

如果表a只有一个文件，大小为120M，但包含几千万的记录，如果用1个map去完成这个任务，肯定是比较耗时的，这种情况下，我们要考虑将这一个文件合理的拆分成多个，这样就可以用多个map任务去完成。

```sql
set mapred.reduce.tasks = 10;
create table a_1 as 
select * from a distribute by rand(123);
```

这样会将a表的记录，随机的分散到包含10个文件的a_1表中，再用a_1代替上面sql中的a表，则会用10个map任务去完成。每个map任务处理大于12M（几百万记录）的数据，效率肯定会好很多.

## 控制reduce数

### 如何确定reduce数？

reduce个数的设定极大影响任务执行效率，不指定reduce个数的情况下，Hive会猜测确定一个reduce个数，基于以下两个设定：

```sql
hive.exec.reducers.bytes.per.reducer;	-- 每个reduce任务处理的数据量，默认是1G
hive.exec.reducers.max;	-- 每个任务最大的reduce数，默认是999
```

计算公式：`N = min(每个任务最大的reduce数, 总输入数据量/每个reduce任务的数据量)`，即：如果reduce的输入总大小不超过1G，那么只有一个reduce任务。比如总大小是9.2G，则会有10个reduce数。

### 调整reduce数

```sql
set hive.exec.reducers.bytes.per.reducer=500000000; (500M)	-- 方法1: 调整每个reduce任务处理数据量是500M
set mapred.reduce.tasks = 15; -- 方法2 手动调整
```

### 什么情况下reduce只有一个

很多时候你会发现任务中不管数据量多大，不管你有没有设置调整reduce个数的参数，任务中一直都只有一个reduce任务；
其实只有一个reduce任务的情况，除了数据量小于`hive.exec.reducers.bytes.per.reducer`参数值的情况外，还有以下原因：

- 没有group by的汇总 `select count(1) from a`
- 用了order by
- 用了笛卡尔积