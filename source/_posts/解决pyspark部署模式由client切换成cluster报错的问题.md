---
title: 解决pyspark部署模式由client切换成cluster报错的问题
date: 2018-12-27 21:56:44
tags: [问题]
categories: [大数据,BigData,Spark]
---

#### 问题

写了一个pyspark的代码，自定义了一些`py`文件`import`进来使用，并且通过`shell`脚本传8个参数，如下：

```shell
#!/usr/bin/env bash
spark-submit \
     --master yarn \
     --deploy-mode cluster \
     --conf spark.shuffle.service.enabled=true \
     --queue xxx \
     --conf spark.dynamicAllocation.enabled=true \
     --conf spark.default.parallelism=1000 \
     --conf spark.sql.shuffle.partitions=1000 \
     --conf spark.sql.broadcastTimeout=7200 \
     --executor-memory 18g \
     --executor-cores 3 \
     --conf spark.blacklist.enabled=true dependencies/test.py $1 $2 $3 $4 $5 $6 $7 $8
```

但是由`--deploy-mode client`切换成`--deploy-mode cluster`之后console上却报如下错误：

<!--more-->

```java
Exception in thread "main" org.apache.spark.SparkException: Application application_1539260237589_6351494 finished with failed status
	at org.apache.spark.deploy.yarn.Client.run(Client.scala:1171)
	at org.apache.spark.deploy.yarn.YarnClusterApplication.start(Client.scala:1539)
	at org.apache.spark.deploy.SparkSubmit$.org$apache$spark$deploy$SparkSubmit$$runMain(SparkSubmit.scala:881)
	at org.apache.spark.deploy.SparkSubmit$.doRunMain$1(SparkSubmit.scala:197)
	at org.apache.spark.deploy.SparkSubmit$.submit(SparkSubmit.scala:227)
	at org.apache.spark.deploy.SparkSubmit$.main(SparkSubmit.scala:136)
	at org.apache.spark.deploy.SparkSubmit.main(SparkSubmit.scala)
```

信息量极少，无法判断和定位问题，通过网上搜索也无法找到答案。

#### 问题定位

很显然，要想定位问题就必须得找到相关的详细日志。我首先想到的就是通过spark 的web UI的查找历史日志，但是通过spark application Id找不到，然后想通过`yarn logs -applicationId id`拉取日志但是报`Log aggregation has not completed or is not enabled.`。

后来看了一下运维给的技术文档，说是还有一种方法是需要到Hadoop集群的RM（`ResourceManager`）节点上查看日志。进入RM的管理页面，搜索对应的的application id，然后点击进入。通过顶部的日志看到：

![pyspark](/images/user_defined/pyspark.png)

*但是注意：*`java.io.FileNotFoundException: File does not exist: hdfs://ns1018/user/jd_ad/ads_model/.sparkStaging/application_1539260237589_6351494/pyspark.zip`不是根本原因。根本原因的从底部的`Logs`链接中看：

![pyspark](/images/user_defined/Logs.png)

点进去之后*不仅`spark_stderr`要看，`spark_stdout`日志也要看*， 在`spark_stdout`日志中看到如下错误：

```java
Traceback (most recent call last):
  File "dmp_id_mapping_invoke.py", line 7, in <module>
    import dmp_id_mapping
ImportError: No module named dmp_id_mapping
```

所以问题是切换成`cluster`模式之后就找不到相应的文件了。

#### 解决问题

找到问题，就容易解决问题，在pyspark中可通过`--py-files dependencies.zip`的方式引入需要`import`的py文件。需要需要`import`的py文件都达在`dependencies.zip`里面。配置如下：

```shell
#!/usr/bin/env bash
spark-submit \
     --master yarn \
     --deploy-mode cluster \
     --conf spark.shuffle.service.enabled=true \
     --queue xxx \
     --conf spark.dynamicAllocation.enabled=true \
     --conf spark.default.parallelism=1000 \
     --conf spark.sql.shuffle.partitions=1000 \
     --py-files dependencies/dependencies.zip \
     --executor-memory 18g \
     --executor-cores 3 \
     --conf spark.blacklist.enabled=true dependencies/test.py $1 $2 $3 $4 $5 $6 $7 $8
```

如何还是不能运行成功，应该是代码中`import`时相关文件的路径涉及不对，排查思路如上，直到问题解决。