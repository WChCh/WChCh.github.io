---
title: 搭建spark本地开发环境
date: 2018-07-06 17:46:00
tags: [Java,Python,开发环境]
categories: [Java,Python,大数据,BigData,Spark]
---

### 准备条件

由于本人的开发机是win10，所以打算在本地搭建spark开发环境，以便于调试。由于目前公司使用的版本是spark-2.1.0，hadoop-2.7.1，所以在spark官网上下载的包为spark-2.1.0-bin-hadoop2.7，在hadoop官网下载hadoop-2.7.1包，然后将这两个包解压。（注意这两个包所在的路径不要有中文或是空格，要不然会出错！），另外还需要去下载对应Hadoop版本的winutils.ext工具，解压后放在~/hadoop-2.7.1/bin目录下，可下载地址为：[https://github.com/steveloughran/winutils](https://github.com/steveloughran/winutils)

<!--more-->

此时可以配置系统环境变量：

1. ['SPARK_HOME'] = "D:\sparkenv\spark-2.1.0-bin-hadoop2.7"
2. ['HADOOP_HOME'] = "D:\sparkenv\hadoop-2.7.1"

当然也可以不用再操作系统中设置，再Idea或是程序中设置也可以。

### 搭建pyspark开发环境

这里使用的是python2.7，因为spark-2.1.0只能是python2.7

1. 首先安装python sdk，文件路径中不要存在空格或是中文字符
2. 在~\Python2.7\Scripts目录下使用pip安装 py4j ： pip install py4j
3. 将~\spark-2.1.0-bin-hadoop2.7\python\pyspark目录拷贝至~\Python2.7\Lib\site-packages目录下作用是编写代码是能过够有spark相关引用的提示。
4. 再IDE中创建python，编写spark应用程序。

如果没有设置'SPARK_HOME'，和'HADOOP_HOME'的环境变量，除了再IDE中单独为每个application设置，还可以再代码中加入：

```
  os.environ['SPARK_HOME'] = "D:\sparkenv\spark-2.1.0-bin-hadoop2.7"
  os.environ['HADOOP_HOME'] = "D:\sparkenv\hadoop-2.7.1"
```

代码中还必须添加([原因](https://blog.csdn.net/u010793236/article/details/73549223))：

```
  # Append pyspark to Python Path
  sys.path.append("D:\spark-1.6.0-bin-hadoop2.6\python")
  sys.path.append("D:\spark-1.6.0-bin-hadoop2.6\python\lib\py4j-0.9-src.zip")
```

接下来就可以开发了。

### 搭建java或是scala开发环境

通过 `System.setProperty("hadoop.home.dir","D:\\\\sparkenv\\\\hadoop-2.7.1" );`来设置HADOOP_HOME路径

通过`SparkConf conf = new SparkConf().setAppName("JavaWordCount").setSparkHome("D:\\\\sparkenv\\\\hadoop-2.7.1\\\\bin").setMaster("local[*]");`设置'SPARK_HOME'