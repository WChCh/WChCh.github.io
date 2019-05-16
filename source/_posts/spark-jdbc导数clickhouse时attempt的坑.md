---
title: spark jdbc导数clickhouse时attempt的坑
date: 2019-04-12 17:48:57
tags: [clickhosue,问题,spark]
categories: [问题,spark,clickhouse,大数据]
---

### 问题

spark 的容错机制会使得spark application失败时尝试重启，从web ui中我们就可以看到其`Attempt ID`大于1。这是个好的机制，但是对于通过jdbc将数据导入clichouse的过程来说却是个不好的体验。因为如果重启的application的话会导致导入的数据重复，使得总的数据量增多，然后zookeeper有数据块的校验机制。

可通过`--conf spark.yarn.maxAppAttempts=1`使得application的重启次数为1，但是又带来可能出现的数据导入不全的情况出现。

看来得寻找一种好的数据导入工具，希望是操作友好，可视化，有监控，易管理，数据不会重复导入的工具。网上看了一下好像NIFI还不错，先调研调研。

### 解决方法

加快导数clickhouse的速度，缩短导入时间，这样就能够大概率上避免spark attempt的出现。具体方法是：

在spark的jdbc properties中

1. 设置合适的batchsize。
2. 设置合适的并发度。
3. 设置合适的内存使用量。

可参考上篇文章：[解决clickhouse批量插入时内存溢出导致docker挂掉的问题](https://wchch.github.io/2019/04/12/%E8%A7%A3%E5%86%B3clickhouse%E6%89%B9%E9%87%8F%E6%8F%92%E5%85%A5%E6%97%B6%E5%86%85%E5%AD%98%E6%BA%A2%E5%87%BA%E5%AF%BC%E8%87%B4docker%E6%8C%82%E6%8E%89%E7%9A%84%E9%97%AE%E9%A2%98/#more)

