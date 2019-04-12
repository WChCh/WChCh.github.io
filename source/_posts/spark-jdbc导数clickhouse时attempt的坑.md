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