---
title: 将数据通过spark从hive导入到Clickhouse
date: 2018-12-20 13:09:13
tags: [clickhosue,olap,MPP,spark]
categories: [olap,BigData,clickhouse,大数据,spark]
---

本文介绍如何通过spark使用JDBC的方式将数据从hive导入到clickhouse中，参考地址为：[https://github.com/yandex/clickhouse-jdbc/issues/138](https://github.com/yandex/clickhouse-jdbc/issues/138)

spark代码`hive2mysql_profile.py`为：

```python
# -*- coding: utf-8 -*-
import datetime
from pyspark.sql import SparkSession
import sys


def sync_profiles(spark, url, driver, yesterday):
    userprofile_b_sql = '''select *  from app.table_test where dt = \'{date}\'  '''.format(
        date=yesterday)
    result = spark.sql(userprofile_b_sql)
    properties = {'driver': driver,
                  'user': 'root',
                  'password': '123456'}

    result.write.jdbc(url=url, table='dmp9n_user_profile_data_bc', mode='append', properties=properties)


if __name__ == '__main__':
    yesterday = (datetime.datetime.now() + datetime.timedelta(days=-1)).strftime("%Y-%m-%d")
    if len(sys.argv) == 2:
        yesterday = sys.argv[1]

    spark = SparkSession.builder \
        .appName("hive2clickhouse") \
        .enableHiveSupport() \
        .getOrCreate()

    url = "jdbc:clickhouse://11.40.243.166:8123/insight"
    driver = 'ru.yandex.clickhouse.ClickHouseDriver'
    sync_profiles(spark, url, driver, yesterday)

```

<!--more-->

submit 任务的脚本如下：

```shell
#!/usr/bin/env bash
spark-submit \
     --master yarn-client \
     --queue xxx \
     --conf spark.akka.frameSize=200 \
     --conf spark.core.connection.ack.wait.timeout=600 \
     --conf spark.rdd.compress=true \
     --conf spark.storage.memoryFraction=0.6 \
     --conf spark.shuffle.memoryFraction=0.4 \
     --conf spark.default.parallelism=720 \
     --conf spark.yarn.executor.memoryOverhead=9216 \
     --executor-memory 18g \
     --executor-cores 3 \
     --num-executors 3 \
     --jars ./clickhouse-jdbc-0.1.28.jar,guava-19.0.jar,httpclient-4.5.2.jar,httpcore-4.4.4.jar,joda-time-2.9.3.jar,lz4-1.3.0.jar \
     --driver-memory 10g \
     --conf spark.driver.maxResultSize=6g \
     --driver-java-options "-XX:MaxPermSize=4096m" ./hive2mysql_profile.py
```

说明**：这六个`clickhouse-jdbc-0.1.28.jar`,`guava-19.0.jar`,`httpclient-4.5.2.jar`,`httpcore-4.4.4.jar`,`joda-time-2.9.3.jar`,`lz4-1.3.0.jar` jar包一定要注意版本号，并且submit spark 任务时将jar包放在对应的目录下。