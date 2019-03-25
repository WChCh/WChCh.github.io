---
title: superset可视化clickhouse安装教程
date: 2019-03-25 18:04:00
tags: [clickhosue,superset]
categories: [clickhouse,可视化]
---

### 官方文档安装

安装环境为centos7，python3.7，python3.7环境配置：https://segmentfault.com/a/1190000015628625#articleHeader1

教程参考：

1. https://superset.incubator.apache.org/installation.html
2. https://zhuanlan.zhihu.com/p/45620490

问题：

```
Could not install packages due to an EnvironmentError: [SSL: DECRYPTION_FAILED_OR_BAD_RECORD_MAC] de
```

使用豆瓣pip源：

```
pip install superset -i https://pypi.douban.com/simple/ --trusted-host pypi.douban.com
```

<!--more-->

问题：

```
Was unable to import superset Error: cannot import name '_maybe_box_datetimelike' from 'pandas.core.common' (/usr/bin/venv/lib/python3.7/site-packages/pandas/core/common.py)
```

解决方法：跟pandas版本有关，卸载掉重装低版本：https://github.com/apache/incubator-superset/issues/6770:

```
pip uninstall pandas
pip list | grep pandas
pip install pandas==0.23.4
```

问题：

```
sqlalchemy.exc.InvalidRequestError: Can't determine which FROM clause to join from, there are multiple FROMS which can join to this entity. Try adding an explicit ON clause to help resolve the ambiguity.
```

解决方法https://github.com/apache/incubator-superset/issues/6977：

```
pip install sqlalchemy==1.2.18
```

连接clickhouse时出现报错：

```
sqlalchemy.exc.NoSuchModuleError: Can't load plugin: sqlalchemy.dialects:clickhouse
```

解决方法：

安装专门支持clickhouse的sqlalchemy ：https://github.com/cloudflare/sqlalchemy-clickhouse

```
pip install sqlalchemy-clickhouse
```

安装pip install mysqlclient  (安装多种数据源客户端以支持多种)出错：

```
OSError: mysql_config not found
```

解决：

```
yum install mysql-devel gcc gcc-devel python-devel
```

superset_config添加mysql配置：

```
mysql://root:123456@10.12.222.212/superset?charset=utf8
```

### docker 镜像部署到k8s

k8s部署问题：

1. 数据库初始化。
2. 镜像选择；
3. namespace选择；
4. 不配置redis的使用。
5. kubectl exec -ti ${your_pods_name} --  初始化，尝试容器初始化
6. docker时区

部署形态：

1. 将superset元数据的存储改为外部的mysql，缓存依赖于redis。
2. k8s中superset为5个pod，其依赖的redis 只为一个pod，只是为了减少有缓存轮空情况，暂无k8s中搭建redis集群打算。
3. 通过`nodePort`方式暴露服务，通过内部域名访问服务。

##### 创建数据库：

```
CREATE DATABASE superset CHARACTER SET utf8 COLLATE utf8_general_ci;
```

##### 制作docker镜像：

dockerfile如下：

```powershell
FROM amancevice/superset:0.29.0rc7

USER root
ADD sources.list /etc/apt/
COPY pip.conf /etc/pip.conf


# Create superset user & install dependencies
RUN pip install --no-cache-dir kylinpy

# 修改docker时区
ENV TZ=Asia/Shanghai
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone

# 添加自定的配置文件
COPY superset_config.py /etc/superset 
USER superset
```

`sources.list`和`pip.conf`和为国内debian源和pip源，superset_config.py为自定义的superset配置文件，都与dockerfile位于同一目录下即可。

superset自定配置文件内容如下：

```
import os

MAPBOX_API_KEY = os.getenv('MAPBOX_API_KEY', '')
CACHE_CONFIG = {
    'CACHE_TYPE': 'redis',
    'CACHE_DEFAULT_TIMEOUT': 300,
    'CACHE_KEY_PREFIX': 'superset_',
    'CACHE_REDIS_HOST': 'redis',
    'CACHE_REDIS_PORT': 6379,
    'CACHE_REDIS_DB': 1,
    'CACHE_REDIS_URL': 'redis://redis:6379/1'}
SQLALCHEMY_DATABASE_URI = 'mysql://user:password@ip:3306/superset?charset=utf8'
SQLALCHEMY_TRACK_MODIFICATIONS = True
SECRET_KEY = 'antCheatingISaSECRET_1234'
ENABLE_PROXY_FIX = True
ROW_LIMIT = 10000
SQL_MAX_ROW = 1000

SUPERSET_WEBSERVER_TIMEOUT = 300
CACHE_DEFAULT_TIMEOUT = 60 * 60 * 24
SQLLAB_TIMEOUT = 300

CSV_EXPORT = {
    'encoding': 'utf-8',
}

```

配置文件中使用了redis缓存，并且使用的外部MySQL。

##### k8s yaml编写

**superset_deployment.yaml**：

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: superset
  namespace: superset
  labels:
    app: superset
spec:
  selector:
    matchLabels:
      app: superset
  replicas: 5
  template:
    metadata:
      labels:
        app: superset
    spec:
      containers:
      - name: superset
        image: path
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8088
```

**superset_service.yaml**：

```
apiVersion: v1
kind: Service
metadata:
  name: superset
  namespace: superset
  labels:
      app: superset
spec:
  type: NodePort
  ports:
    - port: 8088
      targetPort: 8088
      nodePort: 31088
      name: superset
      protocol: TCP
  selector:
    app: superset
```

**redis_deployment.yaml**：

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: superset
  labels:
    app: redis
spec:
  selector:
    matchLabels:
      app: redis
  replicas: 1
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: path
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 6379
```

**superset_service.yaml**：

```
apiVersion: v1
kind: Service
metadata:
  name: superset
  namespace: superset
  labels:
      app: superset
spec:
  type: NodePort
  ports:
    - port: 8088
      targetPort: 8088
      nodePort: 31088
      name: superset
      protocol: TCP
  selector:
    app: superset
```

##### 初始化数据库：

部署到k8s中之后需要初始化数据库，初始化命令如下：

```
kubectl exec -ti ${your_pods_name} -- superset-init
```

之后按照提示操作即可。

##### 仓库登陆问题：

```
Get https://mirror.jd.com/v2/: dial tcp 172.28.217.53:443: connect: connection refused
```

解决：

配置文件`/usr/lib/systemd/system/docker.service`中添加`--insecure-registry=mirror.jd.com`：

```
ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock --insecure-registry=mirror.jd.com
```

或者/etc/docker/daemon.json中添加`"insecure-registries": ["mirror.jd.com"]`：

```
{
"registry-mirrors": ["http://mirror.jd.com"],
"insecure-registries": ["mirror.jd.com"]
}
```

**注意**：域名后面不添加端口。并且这两处不可以同时配置，只能选其一，要不然无法重启docker。

更详细的报错信息可通过系统日志来看：

```
tail -f /var/log/messages
```