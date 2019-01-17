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
                  "socket_timeout": "300000",
                  "rewriteBatchedStatements": "true",
                  "batchsize": "1000000",
                  "numPartitions": "1",
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

`"socket_timeout": "300000"`是为了解决`read time out`异常的问题，当某个分区的数据量较大时会出现这个问题，单位为毫秒。如果是Java语言参考：[https://github.com/yandex/clickhouse-jdbc/issues/159#issuecomment-364423414](https://github.com/yandex/clickhouse-jdbc/issues/159#issuecomment-364423414)

`rewriteBatchedStatements`，`batchsize`, `numPartitions`解决`DB::Exception: Merges are processing signiﬁcantly slower than inserts`问题，原因是批次写入量少，并发多。`batchsize`控制批次写入量，`numPartitions`控制并发数，参考链接：[https://spark.apache.org/docs/latest/sql-data-sources-jdbc.html](https://spark.apache.org/docs/latest/sql-data-sources-jdbc.html)

插入数据时建议大批量，少并发插入，这样当任务完成时，后台也能够及时完成merge，使得数据量一致，而且也不会导致出现别的问题。

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

还有一个需要大家帮忙解答的问题：通过spark导入数据完后发现在在clickhouse中跟hive的中的差了好多，然后不断地count()发现数据不断地增加，直到与hive中的一致，而且这个阶段花的时间也比较长。请大家帮忙解答其中原理。

答案在此[https://github.com/yandex/ClickHouse/issues/542](https://github.com/yandex/ClickHouse/issues/542)：

```
Almost certainly this is the merge process. MergeTree table consists of a number of parts that are sorted by the primary key. Each INSERT statement creates at least one new part. Merge process periodically selects several parts and merges them into one bigger part that is also sorted by primary key. Yes, this is normal activity.
```

并且建议增大merge线程数`background_pool_size`（默认为16）：

```
You could adjust size of thread pool for background operations.
It could be set in users.xml in background_pool_size in /profiles/default/.
Default value is 16. Lowering the size of this pool allows you to limit maximum CPU and disk usage.
But don't set it too low.
```

