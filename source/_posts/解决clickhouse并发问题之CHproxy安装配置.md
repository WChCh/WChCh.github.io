---
title: 解决clickhouse并发问题之CHproxy安装配置
date: 2019-06-14 21:33:52
tags: [clickhosue,olap,MPP，并发，代理]
categories: [olap,BigData,clickhouse,大数据]
---

### 问题

由于新的superset看板查询时每个chart涉及到的底表数据量增加以及并发较多导致clickhouse因为OOM挂掉而返回错误，并且影响到别的看板的使用以及整个系统的稳定性。但是单独查询某个chart是并没有问题，因此需要控制看板查询时的并发度。

### 解决

通过调研以及社区的帮助，可通过代理的方式控制并发度，并且可使用的代理有`[ProxySQL](https://proxysql.com/)`以及专门为clickhouse开发的第三方[CHproxy](https://github.com/Vertamedia/chproxy)。考虑到CHproxy对clickhouse的很多原生配置特性支持的比较好，比如`set`属性相关的（可以控制每个查询所使用的最大内存等）以及负载均衡，并发控制等等很多特性，故选择了CHproxy。

### 安装部署

CHproxy因为是go语言写的，因此安装部署也很简单，也可以通过docker进行部署，具体可以参考文档。启动命令如下：

```shell
./chproxy -config=/path/to/config.yml
```

下面是配置文件，仅供参考：

`config.yml`

```shell
server:
  http:
      listen_addr: ":9092"

      # Networks with reporting servers.
      allowed_networks: ["11.3.0.0/16","172.0.0.0/8","127.0.0.0/8"]

users:
  - name: "proxy-user"
    password: "password"
    to_cluster: "select"
    to_user: "user"
    max_concurrent_queries: 10
    max_execution_time: 3m
    max_queue_size: 120
    max_queue_time: 120s
    cache: "shortterm"
    params: "cron-job"

clusters:
  - name: "select"
    replicas:
      - name: "replica1"
        nodes: ["ip11:8123", "ip21:8123"]
      - name: "replica2"
        nodes: ["ip12:8123", "ip22:8123"]
    heartbeat_interval: 1m
    users:
      - name: "user"
        password: "password"
        max_concurrent_queries: 10
        max_execution_time: 3m
        max_queue_size: 120
        max_queue_time: 120s

caches:
  - name: "shortterm"
    dir: "~/chproxy/cache/dir"
    max_size: 150Mb
    expire: 14400s

param_groups:
    # Group name, which may be passed into `params` option on the `user` level.
  - name: "cron-job"
    # List of key-value params to send
    params:
      - key: "max_memory_usage"
        value: "9000000000"

      - key: "max_bytes_before_external_group_by"
        value: "4500000000"

  - name: "web"
    params:
      - key: "max_memory_usage"
        value: "5000000000"

      - key: "max_columns_to_read"
        value: "30"

      - key: "max_execution_time"
        value: "30"
```

