---
title: spark sql 调优
date: 2018-10-12 17:53:47
tags: [spark-sql,调优]
categories: [大数据,BigData,Spark,Spark-Sql，调优]
---

因为在电商工作所有有机会接触到上百甚至上千亿级的数据，所以在实际工作当中难免会遇到资源配置调优和数据倾斜问题，通过组内
同事以及网上各种教程的帮助，终于解决了一系列问题，达到了了上线标准。感谢组内老大给我这么多的时间让我去学习，去研究，同时
也希望将这个过程记录下来作为以后避免以后遇到类似问题时做重复的工作。

<!--more-->

### 资源配置

由于开始时资源配置不合理，导致出现了各种各样的问题，解决了这些问题之后，配置如下：

```
#!/usr/bin/env bash
spark-submit --deploy-mode client --master yarn \
     --queue root.bdp_jmart_ad.jd_ad_dmp \
     --conf spark.rdd.compress=true \
     --conf spark.sql.shuffle.partitions=10000 \
     --conf spark.yarn.executor.memoryOverhead=9216 \
     --conf spark.network.timeout=1800 \
     --conf spark.sql.broadcastTimeout=1800 \
     --conf spark.executor.heartbeatInterval=20 \
     --conf spark.shuffle.service.enabled=true \
     --executor-memory 27g \
     --executor-cores 3 \
     --num-executors 90 \
     --driver-memory 10g \
     --conf spark.driver.maxResultSize=6g \
     --driver-java-options "-XX:MaxPermSize=4096m" ./xxxxxxx.py
```

当key比较多时，应该适当地调大`--conf spark.sql.shuffle.partitions`的配置，否则可能导致某一个task处理的key比较多，耗时很
长，会出现内存溢出。但是当某些个key自己对应的数据量很大时，即使调大也没用。

当出现广播变量超时或是空指针异常时，可适当调大`--conf spark.sql.broadcastTimeout`这个值。

### spark sql内存方法使用

当数据量比较小时，使用广播变量参与`join`,比如：`result_brand.join(F.broadcast(num_total_bc), ["xxxx"], "inner")`

如果没有特别需求，`groupBy`优先于`Window.partitionBy`，`groupBy`会先局部聚合，再全局聚合。

当实现`not in`查询时，如果`not in`的条件数据极小，可将其`colleact()`到driver端，使用`df = df_total.where(~df_total["lead"].isin(leads_sample)))`
和`df = df_total.where(df_total["lead"].isin(leads_sample)))`实现，`~`表示取反，`leads_sample`为list。可避免使用
`result_brand.join(F.broadcast(leads_sample), ["xxxx"], "left_outer")`还得`dropDuplicates()`行去重导致的shuffle。

### 数据倾斜

解决性能问题最主要的还是解决数据倾斜问题。目前遇到的还是`join`和`groupBy`遇到的数据倾斜问题，网上的解决方案也都一致，我也是参考网上的方案，下面结合代码说明。

#### groupBy导致的数据倾斜

可通过给key加前缀的方式进行两阶段`groupBy`。

第一阶段聚合，首先给相关key添加前缀，再进行`groupBy`:

```python
# 增加前缀
result = result.withColumn("id", F.monotonically_increasing_id() % 1000)
result = result.withColumn("brand_id_order", F.concat_ws("_", "id", "brand_id_order"))
result = result.repartition(10000, "brand_id", "item_third_cate_id", "brand_id_order")
result_brand = result.groupBy("brand_id", "item_third_cate_id", "brand_id_order") \
                      .agg(F.first("brand_name_order", ignorenulls=True).alias("brand_name_order"),
                      F.count("sale_ord_det_id").alias("brand_num")) 
```

这里没有使用随机前缀，而是使用`F.monotonically_increasing_id()`这个固定前缀，利于对比。

第二阶段聚合，去掉相关key的前缀，再进行`groupBy`:

```python
# 移除前缀
result_brand = result_brand.withColumn("brand_id_order", remove_prefix("brand_id_order"))
result_brand = result_brand.groupBy("brand_id", "item_third_cate_id", "brand_id_order") \
                           .agg(F.first("brand_name_order").alias("brand_name_order"),
                           F.sum("brand_num").alias("brand_num"))
```

移除前缀的UDF定义如下：

```python
remove_prefix = F.udf(lambda col: col[col.index('_') + 1:])
```

#### join导致的数据倾斜

首先对数据进行抽样，将有倾斜的key与非倾斜的key拆分，分别进行`join` ，然后再将结果进行`union`。

针对倾斜部分数据，将大表的key添加上随机前缀，小表膨胀n倍，同样的一条数据膨胀后再key上加1~n的前缀进行区分。

```python
# 对数据进行采样
leads_sample = key_sample_c()

# 去掉倾斜key
df_purchase = df_purchase_total.where(~df_purchase_total["lead"].isin(leads_sample))
df_cate_lead = df_cate_lead_total.where(~df_cate_lead_total["lead"].isin(leads_sample))

result_no_skew = df_purchase.join(df_cate_lead, (df_cate_lead.lead == df_purchase.lead), "inner") \
.select("brand_id", "item_third_cate_id", "item_third_cate_id_order",
        "item_third_cate_name_order", "sale_ord_det_id")

# 获取倾斜key
df_purchase = df_purchase_total.where(df_purchase_total["lead"].isin(leads_sample))
df_cate_lead = df_cate_lead_total.where(df_cate_lead_total["lead"].isin(leads_sample))

df_purchase = df_purchase.withColumn("lead", random_prefix(df_purchase["lead"]))

df_cate_lead = df_cate_lead.withColumn("lead", modifyValue(df_cate_lead["lead"]))
df_cate_lead = df_cate_lead.withColumn("lead", F.explode(F.split(df_cate_lead["lead"], ",#")))

result_skew = df_purchase.join(df_cate_lead, (df_cate_lead.lead == df_purchase.lead), "inner") \
.select("brand_id", "item_third_cate_id", "item_third_cate_id_order",
        "item_third_cate_name_order", "sale_ord_det_id")

result = result_no_skew.union(result_skew)
```

采样方法`key_sample_c()`代码如下：

```python
default_fraction = 0.001
real_key_threshold = 1000000
sample_key_threshold = real_key_threshold * default_fraction

def key_sample_c():
    df_purchase = df_purchase.sample(False, default_fraction, seed=0)

    result = df_purchase.join(df_cate_lead, ["lead"], "inner")

    result = result.groupBy("lead").agg(F.count("lead").alias("lead_num"))
    result = result.filter(result["lead_num"] >= sample_key_threshold).select("lead")
    result = result.collect()

    leads = []
    for row in result:
        lead = row["lead"]
        leads.append(lead)

    return leads
```

因为`df_purchase`为大表，所以对其进行采样。采样比例为`default_fraction`，`sample_key_threshold`表示key的过滤阈值，`real_key_threshold`表示对应到真实表中实际的过滤阈值。

增加随机前缀`random_prefix()`方法如下：

```python
random_prefix = F.udf(lambda col: str(random.randint(1, default_range)) + "_" + col)
```

`modifyValue()`方法代码如下：

```python
default_range = 100
def modify_value(value):
    ret_value = "1_" + value
    for i in range(default_range - 1):
        ret_value = ret_value + ",#" + str(i + 2) + "_" + value
    return ret_value


modifyValue = F.udf(modify_value)
```



### 存在问题

解决了数据倾斜的问题之后，如果资源充足的话，时间降到了1.5个小时左右，但还存在一下问题：

1. 经过观察发现，因为中间有dataframe复用的原因，导致每个复用都要从头开始执行，从而产生多个小job，这样的话大表就会被读取很多次，每一次读取的时间都很耗时，因为是大表又无法cache。想过使用中间表的方式减少重复读取次数，但是由于业务较为复杂，暂时没有好的方案。
2. 之所以出现1的问题是因为之前想用`Window.partitionBy`来减少job数量，但是缺出现内存溢出或是内存空间不足的问题，然后采用了链接[NullPointerException in ShuffleExternalSorter.spill()](https://issues.apache.org/jira/browse/SPARK-22517)中题主说的使用多个小job的方法解决。

### 参考链接

1. [Spark性能优化](https://www.iteblog.com/archives/1659.html)
2. (Spark性能优化指南——高级篇)[https://tech.meituan.com/spark_tuning_pro.html]
3. (Spark性能优化之道——解决Spark数据倾斜（Data Skew）的N种姿势)[http://www.jasongj.com/spark/skew/]

