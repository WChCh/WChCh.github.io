---
title: clickhouse ReplicatedMergeTree使用
date: 2019-03-25 18:09:42
tags: [clickhosue,olap,MPP]
categories: [olap,BigData,clickhouse,大数据]
---

#### Nested数据类型使用

待写。

#### CSV http方式灌数命令

```
cat da.csv | curl 'http://10.185.217.47:8123/?user=user&password=password&query=INSERT%20INTO%20table%20FORMAT%20CSV'
```

#### 库、表操作

库，表的创建，删除等操作加上`on cluster cluster1`说明，只在一个节点上操作即可实现集群同步。

#### 相关问题

问题删除分区时：

```
Password required for user default., e.what() = DB::Exception."
```

解决[https://github.com/yandex/ClickHouse/issues/4762](https://github.com/yandex/ClickHouse/issues/4762)：

原因：

```
Generally, when you do some DDL on non-leader replica it forwards the request to a leader, and during that forwarding "default" passwordless user was used.
从leader replica删除。
```

解决：选择所有的主副本删除即可，zookeeper会自动同步删除子副本的数据。

<!--more-->

问题：

```
Traceback (most recent call last):
  File "/data0/home/ads_model/wangchuanchao/anti_cheating/test/hive2clickhouse/./h2c_anti_cheating_log.py", line 40, in <module>
    sync_profiles(spark, shard_index)
  File "/data0/home/ads_model/wangchuanchao/anti_cheating/test/hive2clickhouse/./h2c_anti_cheating_log.py", line 27, in sync_profiles
    result_df.write.jdbc(url=url, table=table_ch_local, mode='append', properties=properties)
  File "/software/servers/tyrande/jd_ad/spark/python/lib/pyspark.zip/pyspark/sql/readwriter.py", line 765, in jdbc
    self._jwrite.mode(mode).jdbc(url, table, jprop)
  File "/software/servers/tyrande/jd_ad/spark/python/lib/py4j-0.10.4-src.zip/py4j/java_gateway.py", line 1133, in __call__
    answer, self.gateway_client, self.target_id, self.name)
  File "/software/servers/tyrande/jd_ad/spark/python/lib/pyspark.zip/pyspark/sql/utils.py", line 79, in deco
    raise IllegalArgumentException(s.split(': ', 1)[1], stackTrace)
IllegalArgumentException: u"Can't get JDBC type for struct<hit_policy:array<string>,refund:int,experiment_group_id:int,gmv:double,anti_time:bigint,refund_time:bigint,more_filter:int,is_spam:int>"
```

https://stackoverflow.com/questions/50201887/java-lang-illegalargumentexception-cant-get-jdbc-type-for-arraystring

**未解决**。

JDBC灌数问题：

```
2019.03.25 13:06:14.364337 [ 37531 ] <Error> executeQuery: Code: 27, e.displayText() = DB::Exception: Cannot parse input: expected \t before: \\N\t\\N\t\\N\t\\N\t\\N\t\\N\t\\N\t\\N\t\t\\N\t\\N\t\\N\t59.63.206.227\t\\N\t\\N\t\\N\t\\N\t\\N\t\\N\t\\N\t\\N\t\\N\t\\N\t\\N\t\\N\t\\N\t\\N\t2019-03-21\n\\N\t8c14467d237e48f39a2ba670fa41657d\t\\N\t\\N\t\\N\t\\N\t\\N\t\\N\t\\N\t\\N: (at row 1)

Row 1:
Column 0,   name: sid,              type: String,  parsed text: "<BACKSLASH>N"
Column 1,   name: click_id,         type: String,  parsed text: "78530fe4367d45c3a37f2cc81f49b354"
Column 2,   name: ad_spread_type,   type: Int32,   ERROR: text "<BACKSLASH>N<TAB><BACKSLASH>N<TAB><BACKSLASH>N<TAB><BACKSLASH>" is not like Int32

, e.what() = DB::Exception (from [::ffff:10.198.145.69]:21814) (in query: INSERT INTO ad_base2_click_local ("sid","click_id","ad_spread_type","ad_traffic_group","ad_traffic_type","ad_plan_id","last_pos_id","last_click_id","sku_id","act_price","advertise_pin","user_pin","user_id","device_id","user_ip","device_type","retrieval_type","keyword","day","click_time","jda_time","is_bill","click_ip","ua","dsp_posid","behaviour","anti_info","charge_data","utm_term","dt")  FORMAT TabSeparated), Stack trace:

```

用特殊标记 ([NULL](https://clickhouse.yandex/docs/zh/query_language/syntax/)) 表示"缺失值"，可以与 `TypeName` 的正常值存放一起。例如，`Nullable(Int8)` 类型的列可以存储 `Int8` 类型值，而没有值的行将存储 `NULL`。

**注意**：如果是在别的机器上通过jdbc导入，出此错误是看到的可能会是乱码，可登陆到clickhouse所在机器查看服务非乱码报错的信息，以更快速定位问题。

CSV灌数问题：

```
cat da.csv | curl 'http://10.185.217.47:8123/?user=user&password=password&query=INSERT%20INTO%20table%20FORMAT%20CSV'
Code: 27, e.displayText() = DB::Exception: Cannot parse input: expected , before: hit_policy\\":[],\\"refund\\":0,\\"experiment_group_id\\":0,\\"ce\\":0,\\"fake: (at row 1)

Row 1:
Column 0,   name: sid,              type: String,  parsed text: "571"
Column 1,   name: click_id,         type: String,  parsed text: "ba3aa5ce-e7d9-4fa2-9f92-5c08224d178f"
Column 2,   name: ad_spread_type,   type: Int32,   parsed text: "1"
Column 3,  name: anti_info,        type: String,  parsed text: "<DOUBLE QUOTE>{<BACKSLASH><DOUBLE QUOTE>"
ERROR: There is no delimiter (,). "h" found instead.
```

解决方法：

clickhouse遵守csv文件格式规范，请注意csv的字符转义规范，比如双引号中嵌套双引号，通常情况下是：`"\""`，csv规范下是`""""`，等到的效果都是`"""`。