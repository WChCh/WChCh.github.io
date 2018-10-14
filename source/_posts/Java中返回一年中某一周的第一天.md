---
title: Java中返回一年中某一周的第一天
date: 2017-11-25 17:13:00
tags: [Java,Calendar,问题,后台]
categories: [Java,Calendar,问题,后台]
---

实现这样一个功能，根据前端的日历组件向后台传回某一年的第几周的开始日期和结束日期，日历组件的展示的日历周是以周日开始周六结束。
但是当直接设置Java中Calendar对象中的年数和周数时，返回这一周第一天（周日）的日期总是对不上。后来经过同事的帮忙总与解决了，代码如下：

```java
// 获取某一周的起始时间
	public static Calendar getFirstDayOfWeek(int year, int week) {

		Calendar firDay = Calendar.getInstance();

		//firDay.setFirstDayOfWeek(Calendar.SUNDAY);
		// 先滚动到该年
		firDay.set(Calendar.YEAR, year);
		// 滚动到周
		firDay.set(Calendar.WEEK_OF_YEAR, week - 1);
		
		firDay.set(Calendar.DAY_OF_WEEK, 0);
		//1天
		firDay.add(Calendar.DAY_OF_YEAR, +1);
		return firDay;
	}
```