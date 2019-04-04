---
title: sparkSql操作hive表的PlainValuesDictionary问题
date: 2019-04-04 09:25:50
tags: [spark-sql,hive]
categories: [大数据,BigData,Spark,Spark-Sql，问题]
---

操作hive表时出现一下报错：

```python
org.apache.parquet.column.values.dictionary.PlainValuesDictionary$PlainIntegerDictionary
```

原因：parquet文件中某些列的数据类型不一致。

例如这个表的某一列的schema类型为string，但是这一列的数据是从不同的数据源插入的，但是某些数据源的类型是int，却也能插入成功，所以当操作此表并且包含此列时，就可能会出现以上问题。

解决：

不同类型的数据插入表时，强制转化成与表schema一致的类型。