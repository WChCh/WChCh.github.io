---
title: '解决OutOfMemoryError: Direct buffer memory问题'
date: 2019-04-09 16:32:02
tags: [spark-sql,调优]
categories: [大数据,BigData,Spark,Spark-Sql，调优]
---

#### 问题

近日处理一些大数据时老是出现`OutOfMemoryError: Direct buffer memory`的问题，一开始以为是数据倾斜问题，然后使用拆分倾斜key分别join再union的方法处理数据倾斜。后来测试发现，反而是非倾斜部分的数据进行join时出现了此问题。

#### 实验过程

我做了些实验：

1. 大表`column1`中`-`和`notset`字符串的量分别为8.5亿和2.8亿，占了大约总量的二分之一。
2. 这两张表中个表我只取`column1`这个字段，并且根据`column1` groupby 之后cont()为num，再将这两张表的结果进行join，并且增加列为表1的num乘以
   表2的num的结果，即为两张原始表join后的数量。结果发现前三数量最大的为16780380，255084，147246，无`-`或是字符串`notset`。
3. 小表table1中没有`column1`为 `-`或是字符串`notset`， 同样这两个字符串也不会再步骤2中出现。也就是`-`和字符串`notset`在left join中起不到任何作用，只会在shuffle是占用大量空间。
4. 通过观察web ui 中的sql 标签页，发现都是大表与大表的“SortMergeJoin”。
5. 因为左连接左小表table1的`column1`中没有`-`和字符串`notset`，在读取右大表直接过滤掉`column1`中含`-`和字符串`notset`的列，至此实验通过，不再报`OutOfMemoryError: Direct buffer memory`的问题。

<!--more-->

#### 原因分析

根据“SortMergeJoin”原理，我认为是：

虽然小表table1中没有`-`或是字符串`notset`，但是仍会将大表table2中的`-`和字符串`notset`  shuffle到某些分区中，因为量大这样可能导致内存溢出。


所以好的优化方法是：

1. 左关联中，将左表中没有的key，但是在右表中量又是特别大的提前从右表中剔除掉。

一个新的分析数据的技巧：
对某一些key join之后再根据数量排序，可以参考实验步骤1。这样可以减少占用的数据占用内存的空间。

#### 处理数据倾斜demo

下面是处理数据倾斜的代码，具体使用说明可参考另一篇文章[spark sql 调优](https://wchch.github.io/2018/10/12/spark-sql-%E8%B0%83%E4%BC%98/)。

```python
day_before_ystd = '2019-04-02'
spark = SparkSession.builder \
        .appName("m04_ord_data") \
        .enableHiveSupport() \
        .getOrCreate()

default_fraction = 0.001
real_key_threshold = 100000
sample_key_threshold = real_key_threshold * default_fraction
default_range = 100


def key_expansion(value):
    ret_value = "1_" + value
    for i in range(default_range - 1):
        ret_value = ret_value + ",#" + str(i + 2) + "_" + value
    return ret_value

key_expansion = F.udf(key_expansion)

def sample_keys(df_click_order, df_behavior):
    # 对大表进行采样
    df_behavior_sample = df_behavior.sample(False, default_fraction, seed=0)

    result = df_click_order.join(df_behavior_sample, df_click_order.left_key == df_behavior_sample.right_key, 'inner')
    result = result.groupBy(result.left_key).agg(F.count(result.left_key).alias("key_count"))
    result = result.filter(result["key_count"] >= sample_key_threshold).select(result.left_key)
    result = result.collect()

    # 获取倾斜的key
    key_ids = []
    for row in result:
        key_id = row["left_key"]
        key_ids.append(key_id)

    return key_ids

random_prefix = F.udf(lambda col: str(random.randint(1, default_range)) + "_" + col)


def behavior_sequence_gen(sql_behavior,sql_click_order):
    df_click_order = spark.sql(sql_click_order)
    df_behavior = spark.sql(sql_behavior)
    keys_sample = sample_keys(df_click_order, df_behavior)
    df_click_order_no_skew = df_click_order.where(~df_click_order["left_key"].isin(keys_sample))
    df_behavior_no_skew = df_behavior.where(~df_behavior["right_key"].isin(keys_sample))
    result_no_skew = df_click_order_no_skew.join(df_behavior_no_skew,
                                                 df_click_order_no_skew.left_key == df_behavior_no_skew.right_key,
                                                 'left_outer').select("column1", "column2")
    df_click_order_skew = df_click_order.where(df_click_order["left_key"].isin(keys_sample))
    df_behavior_skew = df_behavior.where(df_behavior["right_key"].isin(keys_sample))
    df_behavior_skew = df_behavior_skew.withColumn("right_key", random_prefix(df_behavior_skew["right_key"]))
    df_click_order_skew = df_click_order_skew.withColumn("left_key", key_expansion(df_click_order_skew["left_key"]))
    df_click_order_skew = df_click_order_skew.withColumn("left_key",
                                                         F.explode(F.split(df_click_order_skew["left_key"], ",#")))
    result_skew = df_click_order_skew.join(df_behavior_skew,
                                           df_click_order_skew.left_key == df_behavior_skew.right_key, 'left_outer').select("column1", "column2")
    df_result = result_no_skew.union(result_skew)
    df_result.write.mode("append").partitionBy("dt").saveAsTable("table")
    
if __name__ == '__main__':
    # 将两张表中需要join的列分别多添加left_key和right_key，这样的话后面再获取column1列时就不需要再去前缀
    sql_table1 = '''select column1, column2, column1 as right_key from table1 
        where dt = '{date}' and column1 is not null and length(column1) != 0 and column1 != '-' and column1 != 'notset' 
        '''.format(date=day_before_ystd)
    sql_table2 = '''select *, column1 as left_key from table2 where dt = '{date}' and column1 is not null and length(column1)!=0 '''.format(date=day_before_ystd)
    behavior_sequence_gen(sql_table1, sql_table2)
```

