---
title: 更为灵活的编程式sql：superset中jinja template的使用
date: 2019-05-13 20:03:12
tags: [使用,superset]
categories: [使用,可视化]
---

superset中仅仅是对数据表的操作很多时候还是没法满足我们的数据展示需求，因此superset提供了jinja template的方式让我们更为灵活地自定义sql语句。

#### 使用说明

通过jinja中文文档http://docs.jinkan.org/docs/jinja2/templates.html，我们可以了解jinja template如何使用，包括分隔符。（前者用于执行诸如 for 循环 或赋值的语句，后者把表达式的结果打印到模板上），filters，条件判断，宏等。

superset关于jinja template的说明：https://superset.incubator.apache.org/sqllab.html。其中superset提供了`filter_values`和`url_param`两个jinja方法，其中`filter_values`更为重要。`filter_values`可以接收filter_box这个chart的值，并且以列表的方式输出。

superset还提供了一些操作函数，与python的import层级一一对应，比如：

- `time`: `time`
- `datetime`: `datetime.datetime`
- `uuid`: `uuid`
- `random`: `random`
- `relativedelta`: `dateutil.relativedelta.relativedelta`

还可以使用jinja内置过滤器并且通过管道的方式使用：http://docs.jinkan.org/docs/jinja2/templates.html#id21。

<!--more-->

#### 例子

以下是在SQLLab中使用jinja template的例子：

```python
SELECT COUNT(DISTINCT userId)/
  (SELECT COUNT(DISTINCT userId)
   FROM AppUsageFactSuperset
   WHERE 0=0
   {% set filters = form_data.get('filters') %}
   
   {% if filters %}
		{% for eachFilter in filters %}
			{% if eachFilter %}
				{% set column_name = eachFilter.get('col') %}
				{% set values = eachFilter.get('val') %}
				{% set operator = eachFilter.get('op') %}
				{% set joined_values = "'" + "','".join(values) + "'" %}
				
				AND {{ column_name }} {{ operator }} ( {{ joined_values }} )
			
		   {% endif %}
		   {% endfor %}
		   {% endif %}
   ) AS `Reach_Temp`
FROM smartmeter_pbi_dot11.`AppUsageFactSuperset`
WHERE 0 = 0
{% set filters = form_data.get('filters') %}
   
   {% if filters %}
		{% for eachFilter in filters %}
			{% if eachFilter %}
				{% set column_name = eachFilter.get('col') %}
				{% set values = eachFilter.get('val') %}
				{% set operator = eachFilter.get('op') %}
				{% set joined_values = "'" + "','".join(values) + "'" %}
				
				AND {{ column_name }} {{ operator }} ( {{ joined_values }} ) 
			
		   {% endif %}
		   {% endfor %}
		   {% endif %}
ORDER BY `Reach_Temp` DESC
LIMIT 50000
```





```python
{% set first_day = (datetime.now() + relativedelta(days=-1)).strftime("%Y-%m-%d") %}
{% set date_time = filter_values('date_time')[-1] %}
{% set date_time=(date_time if date_time|trim|length > 0 else first_day) %}

{% set advertise_pins = filter_values('advertise_pin') %}
{% set ad_group_ids = filter_values('ad_group_id') %}
{% set ad_plan_ids = filter_values('ad_plan_id') %}
{% set campaign_types = filter_values('campaign_type') %}
{% set hours = filter_values('hour') %}

{% macro conditions(date_time, advertise_pins, ad_group_ids, ad_plan_ids, campaign_types, hours) -%}
    where
    dt='{{date_time}}' 
    {% if advertise_pins|length > 0 %}
        AND advertise_pin in ( {{ "'" + "','".join(filter_values('advertise_pin')) + "'" }} )
    {% endif %}
    {% if ad_group_ids|length > 0 %}
        AND ad_group_id in ( {{ "'" + "','".join(filter_values('ad_group_id')) + "'" }} )
    {% endif %}
    {% if ad_plan_ids|length > 0 %}
        AND ad_plan_id in ( {{ "'" + "','".join(filter_values('ad_plan_id')) + "'" }} )
    {% endif %}
    {% if campaign_types|length > 0 %}
        AND campaign_type in ( {{",".join(filter_values('campaign_type'))}} )
    {% endif %}
    {% if hours|length > 0 %}
        AND hour in ( {{ "'" + "','".join(filter_values('hour')) + "'" }} )
{% endif %}
{%- endmacro %}

select
user_ipc,
count(1) as num,
count(1)/(select count(1) from anti.ad_base2_click {{conditions(date_time, advertise_pins, ad_group_ids, ad_plan_ids, campaign_types, hours)}} ) as ratio
from anti.ad_base2_click
{{conditions(date_time, advertise_pins, ad_group_ids, ad_plan_ids, campaign_types, hours)}}
group by user_ipc
order by num desc
limit 10
```

```python
{% set first_day = (datetime.now() + relativedelta(days=-2)).strftime("%Y-%m-%d") %}
{% set date_time = filter_values('date_time')[-1] %}

{% set date_time= ((datetime.strptime(date_time, "%Y-%m-%d") + relativedelta(days=-1)).strftime("%Y-%m-%d")  if date_time|trim|length > 0 else first_day) %}
```

