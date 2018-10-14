---
title: csv文件导入hive表
date: 2018-08-29 17:49:22
tags: [csv,hive]
categories: [hive,大数据,BigData,Spark]
---

一个典型的hive如下：

```
CREATE EXTERNAL TABLE `model.unbounded_dmp_ad_tables_jd2_imei_test`(
  `jd_pin` string COMMENT '京东pin',
  `_c1` string COMMENT 'column1 name',
  `_c2` string COMMENT 'column2 name',
  `_c3` string COMMENT 'column3 name')
ROW FORMAT DELIMITED
  FIELDS TERMINATED BY '\u0001'
  LINES TERMINATED BY '\n'
STORED AS INPUTFORMAT
  'org.apache.hadoop.mapred.TextInputFormat'
OUTPUTFORMAT
  'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
LOCATION
  '/user/jd_ad/ads_model/P_and_G/result/mapped/tables_jd2_imei'
TBLPROPERTIES (
  'mart_name'='jd_ad',
  'transient_lastDdlTime'='1476148520',
  'skip.header.line.count'='1');
```

重点说明如下：

1. 因为每个csv文件都带有header，所以使用`'skip.header.line.count'='1'`来跳过header行。
2. 字段间分隔符使用'\u0001',使用pyspark时分隔符为sep="\x01"