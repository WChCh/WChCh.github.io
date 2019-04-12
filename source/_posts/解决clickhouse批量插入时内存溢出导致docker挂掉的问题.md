---
title: 解决clickhouse批量插入时内存溢出导致docker挂掉的问题
date: 2019-04-12 11:20:53
tags: [clickhosue,olap,MPP]
categories: [olap,BigData,clickhouse,大数据]
---

### 问题

为了加快导入数据的速度我将针对每个分片的导入并行度都设置为3，如下配置：

```python
properties_4_click_order_behavior = {'driver': 'ru.yandex.clickhouse.ClickHouseDriver',
              "socket_timeout": "300000",
              "rewriteBatchedStatements": "true",
              "batchsize": "1000000",
              "numPartitions": "3",
              'user': CONF_CONSTANT.user,
              'password': CONF_CONSTANT.password}
```

之后各个分片就报出了`connection refused`的问题，然后运维那边说是内存溢出了，我们的每个分片的内存只有24G。当然，当`numPartitions`为1的时候是没问题的。

我还是是希望能够加快能够加快导入速度，1个并行插入效率确实低，而且还有一个让我比较疑惑的地方是，我们有一个只有3个容器，配置为8核，32G，只有1副本，但是我令`numPartitions`为12，`batchsize`为20000000都没有问题。

### 初步解决

这里得感谢，QQ群`Clickhouse牛人帮`的几位大牛的帮助，我通过翻聊天记录查看到与我遇到类似问题的，并且有了解决方法，就是通过设置`max_memory_usage`和`max_bytes_before_external_group_by`来控制内存使用，按照官方文档建议，`max_bytes_before_external_group_by`为`max_memory_usage`二分之一。既然容器的内存只有24G，那我就设置`max_memory_usage`为20G好了，设置如下：

```
properties_4_click_order_behavior = {'driver': 'ru.yandex.clickhouse.ClickHouseDriver',
              "socket_timeout": "300000",
              "rewriteBatchedStatements": "true",
              "batchsize": "1000000",
              "numPartitions": "3",
              'user': "user",
              'password': "password",
              'max_memory_usage': "20000000000",
              'max_bytes_before_external_group_by': "10000000000"}
```

一开始我设置`numPartitions`为8，但是仍然出现`connection refused`的问题，后来调整成3就好了。为什么这样呢？



<!--more-->



我想能不能先看一下每批次insert时占用内存是多少，然后通过如下SQL：

```mysql
select written_rows, memory_usage,  query, Settings.Names, Settings.Values  from system.query_log where user='ads_model_rw' and client_name!='ClickHouse client'  limit 20
```

发现`memory_usage`大概为4G左右，所以这样的话3个并行insert是可以的（如果有能好的评估方法，请指教）。

### 疑惑

之前那个只有1副本的3分片的集群(8cores，32G内存)为什么就能比这个2副本15分片的集群(32cores，24G内存)承受更多的insert并发度呢？是否需要更改哪些配置参数？更改`max_memory_usage`之前，1副本的3分片的集群的值大概为64G，2副本15分片的集群的值大概为99G。求解答！



