---
title: pyspark中删除dataframe中的嵌套列
date: 2019-03-25 18:07:46
tags: [clickhosue,olap,pyspark]
categories: [olap,BigData,clickhouse,大数据]
---

hive表中有某一列是`struct`类型，现在的需求是将这个`struct`类型中的某一子列抽取出来，并且转换成字符串类型之后，添加成与`struct`类型的列同一级别的列。

然后网上搜了一下答案，发现使用scala操作子列很方便，但是我们组使用语言还是python，然后搜到此方法方法：drop nested columns [https://stackoverflow.com/questions/45061190/dropping-nested-column-of-dataframe-with-pyspark/48906217#48906217](https://stackoverflow.com/questions/45061190/dropping-nested-column-of-dataframe-with-pyspark/48906217#48906217)。我参照此方法针对我的需求做了修改。

`exclude_nested_field`方法中将去掉不需要的`field.name`，及其对应的`StructType`包装成的`StructField`，这样从schema上看就是移除了某一子列。

```python
def exclude_nested_field(schema, unwanted_fields, parent=""):
    new_schema = []
    for field in schema:
        full_field_name = field.name
        if parent:
            full_field_name = parent + "." + full_field_name
        if full_field_name not in unwanted_fields:
            if isinstance(field.dataType, StructType):
                inner_schema = exclude_nested_field(field.dataType, unwanted_fields, full_field_name)
                new_schema.append(StructField(field.name, inner_schema))
            else:
                new_schema.append(StructField(field.name, field.dataType))
    return StructType(new_schema)
```

<!--more-->

`exclude_nested_field_name`方法中将去掉不需要的`StructField`中的`field.name`，只保留对应的`StructType`。

```python
def exclude_nested_field_name(schema, unwanted_field_name, parent=""):
    new_schema = []
    for field in schema:
        full_field_name = field.name
        if parent:
            full_field_name = parent + "." + full_field_name
        if full_field_name==unwanted_field_name:
            if isinstance(field.dataType, StructType):
                print(field.dataType)
                inner_schema = exclude_nested_field(field.dataType, unwanted_field_name)
                return  inner_schema
        elif isinstance(field.dataType, StructType):
            inner_schema = exclude_nested_field(field.dataType, unwanted_field_name, full_field_name)
            new_schema.append(StructField(field.name, inner_schema))
        else:
            new_schema.append(StructField(field.name, field.dataType))
    return StructType(new_schema)
```

使用示例：

```python
# charge_data为struct类型，过滤掉charge_data这个field name对应的StructType中的details StructField
new_schema = exclude_nested_field(anti_df.select("charge_data").schema, ["charge_data.details"])
# 过滤掉最外层charge_data这个field name对应的StructField中的StructType
new_schema = exclude_nested_field_name(new_schema, "charge_data")

schema_details = exclude_nested_field_name(anti_df.select("charge_data.details").schema, "details")
anti_df = anti_df.withColumn("json_charge_data", F.to_json(anti_df.charge_data))
# 用新的schema和json恢复成dataframe，并根据schema移除掉相关的子列。
anti_df = anti_df.withColumn("charge_data", F.from_json("json_charge_data", new_schema))
anti_df = anti_df.withColumn("charge_data_details", F.to_json(F.from_json("json_charge_data", schema_details))).drop("json_charge_data")
```