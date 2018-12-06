---
title: python画图工具matplotlib使用
date: 2018-12-06 17:25:21
tags: [python,matplotlib,工具,画图]
categories: [python,工具,画图]
---

#### 前言

对于数据画图本人毕业之前还是比较喜欢用matlab，但现在工作后本地没有安装matlab软件，所以只能打算用python来画图。上网搜了一下，发现python中使用matplotlib来画图很受欢迎，加上刚好有画图的需要，所以打算试一下。

需求：现有2017一年的数据，时间间隔为1小时，所以数据量大概为 24 * 365。由于图片尺寸不能过大，横坐标显示所有值的话会密密麻麻重重叠叠，所以横坐标只能按月份显示，但纵坐标的数据必须都得显示。

<!--more-->

#### 体验

文字就不多说了，直接上会说话的代码。

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-
import datetime
import matplotlib.pyplot as plt
import matplotlib as mpl
import matplotlib.dates as mdate

if __name__ == '__main__'
    filepost = open('E:\\post-2017.txt', 'r')

    WL = []
    for line_post in filepost.readlines():
        timePost = line_post.strip().split(",")
        x.append(datetime.datetime.strptime(timePost[0], '%Y-%m-%d %H:%M:%S')) # 第一列为 %Y-%m-%d %H:%M:%S 格式的时间，注意datetime是有时分秒的，date没有时分秒，如果你的数据对应到时或分或秒的话返回date（）将是画图的数据不一致。
        WL.append(float(timePost[1])) # 必须转换为数值型

    filepost.close()

    plt.figure(figsize=(96, 20))  # 设置图的大小，应用于plt.plot()之前
    # plt.rcParams['savefig.dpi'] = 600  # 图片像素
    # plt.rcParams['figure.dpi'] = 600  # 分辨率
    # plt.yticks(np.arange(2.5, 3, 0.03)) # 设置y轴刻度
    plt.xlabel("Hours")
    plt.ylabel('Value')
    ax = plt.gca()
    # xlocator = mpl.ticker.LinearLocator(12) # 设置x轴显示数值的个数
    # xlocator = mpl.ticker.MultipleLocator(6 * 24) # 设置x轴显示数值的间隔（非时间类型）
    # ax.xaxis.set_major_locator(xlocator)
    # ylocator = mpl.ticker.MultipleLocator(6 * 24)# 设置y轴显示数值的间隔
    # ax.yaxis.set_major_locator(ylocator)
    plt.xticks(rotation=45) # x轴数值旋转45度
    plt.title("Chart of post-2017")
    plt.plot(x, WL, label='Corrected-WL/Pressure', color='r', alpha=0.5)
    plt.legend(loc='upper left')
    ax.xaxis.set_major_formatter(mdate.DateFormatter('%Y-%m'))  # 设置x轴时间标签显示格式
    ax.xaxis.set_major_locator(mdate.MonthLocator())  # 设置x轴时间标签显示间隔，按月显示
    plt.savefig("D:\\post-2017.svg")
    # plt.show()
```

