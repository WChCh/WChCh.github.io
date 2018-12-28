---
title: 解决hive表小文件过多问题
date: 2018-09-06 17:51:45
tags: [小文件合并,hive]
categories: [hive,大数据,BigData,Spark]
---

## 背景
前些时间，运维的同事反应小文件过多问题，需要我们去处理，所以想到是以何种手段去合并现有的小文件。我们知道Hadoop需要在namenode维护文件索引相关的metadata，所以小文件过多意味着消耗更大的内存空间。

## 过程

经过网上的调研发现通过hive表使用orc格式进行存储能够通过`concatenate`命令对分区进行小文件合并，并且能够节省80%以上的存储空间，真是喜闻乐见！

<!--more-->

本文不再详细介绍orc，text，rc，parquet各种对比，具体可见网上相关文章，下面只是以举例为主。

创建一个orc hive 表：

```mysql
CREATE EXTERNAL TABLE `app.example_orc`(
  `timestamp` string COMMENT '时间戳',
  `city` string)
PARTITIONED BY (
  `dt` string)
STORED AS ORC
LOCATION
  'hdfs://xxxxxxxx/test'
TBLPROPERTIES (
  'mart_name'='xxxx',
  'transient_lastDdlTime'='1476148520');
```

从别的表导数据到此表的`20180505`分区：

```mysql
INSERT INTO TABLE app.example_orc partition(dt="20180505",dt="xxxxx"...) select timestamp, city from app.nielsenid_device2pin_unmapped where dt="20180505"
```

使用`concatenate`命令针对`20180505`分区进行小文件合并：

```mysql
alter table app.example_orc partition (dt="20180505") concatenate;
```

 

不足点：

1. 使用`concatenate`命令合并小文件时不能指定合并后的文件数量，虽然可以多次执行该命令，但显然不够优雅。当多次使用`concatenate`后文件数量不在变化，这个跟参数`mapreduce.input.fileinputformat.split.minsize=256mb`的设定有有有关，可设定每个文件的最小size，具体间链接4；
2. 只能针对分区使用`concatenate`命令。

参考连接：

1. [https://mapr.com/blog/what-kind-hive-table-best-your-data/](https://mapr.com/blog/what-kind-hive-table-best-your-data/)
2. [https://datashark.academy/how-to-avoid-small-files-problem-in-hadoop/](https://datashark.academy/how-to-avoid-small-files-problem-in-hadoop/)
3. [https://blog.csdn.net/yu616568/article/details/51188479](https://blog.csdn.net/yu616568/article/details/51188479)
4. [https://community.hortonworks.com/questions/212611/hivepartitionssmall-filesconcatenate.html](https://community.hortonworks.com/questions/212611/hivepartitionssmall-filesconcatenate.html)