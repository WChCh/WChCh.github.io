---
title: 搭建clickhouse监控
date: 2019-06-15 18:34:37
tags: [clickhosue,olap,MPP，监控，代理]
categories: [olap,BigData,clickhouse,大数据]
---

### 原由与方案

为了更好地优化clickhouse的性能，需要对clickhouse集群进行监控。网上很多监控方案都是clickhouse + grafana + prometheus，因此打算使用此方案。

要想使用prometheus就得先安装exporter，clickhouse有第三方提供的[clickhouse_exporter](https://github.com/f1yegor/clickhouse_exporter/blob/master/clickhouse_exporter.go)，也有容器版本，并且提供了grafana的dashboard版本： https://grafana.net/dashboards/882。由于我们在集群中使用了代理CHproxy，但CHproxy也同时实现了exporter的功能，并且提供了更多的特性以及grafana dashboard模板https://github.com/Vertamedia/chproxy/blob/master/chproxy_overview.json，所以我们也就直接使用。

### 安装部署

我们使用了docker进行部署，并且使用docker-compose进行编排，并且将配置文件和重要数据挂载到宿主机。

docker-compose如下：

```shell
version: '3'
services:
  prometheus:
    image: prom/prometheus:latest
    restart: always
    network_mode: host
    user: root
    container_name: prometheus
    ports:
      - "9090:9090"
    depends_on:
      - chproxy
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - ./prometheus-data:/prometheus
  grafana:
    image: grafana/grafana:latest
    restart: always
    network_mode: host
    user: root
    container_name: grafana
    ports:
      - "3000:3000"
    depends_on:
      - prometheus
    volumes:
      - ./grafana-data/var/lib/grafana:/var/lib/grafana
      - ./grafana-data/etc/grafana:/etc/grafana

  chproxy:
    image: tacyuuhon/clickhouse-chproxy:1.13.2
    restart: always
    network_mode: host
    container_name: chproxy
    ports:
      - 9092:9092
    volumes:
      - ./config.yml:/opt/config.yml

```



CHproxy的配置文件`config.xml`，省略，参考上篇文章。

prometheus的配置文件`prometheus.xml`如下：

```shell
# my global config
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
    - targets: ['localhost:9090']

  - job_name: 'clickhouse-chproxy'

    # 覆盖全局的 scrape_interval
    scrape_interval: 10s

    static_configs:
    - targets: ['localhost:9092']

```

可通过`http://ip:9090/targets`查看prometheus配置文件中配置的job是否成功。

**请注意：**

1. 因为docker-compose中使用的网络模式为`host`，所以在prometheus的配置文件中ip地址都写为`localhost`，并且要填上对应容器的端口。
2. grafana容器可将`/var/lib/grafana`和`/etc/grafana`目录拷贝出来放到宿主机上并且重新挂载到容器中这样的话删除并且重启容器时不会导致数据丢失。docker-compose中grafana和prometheus容器可能需要将`user`设为`root`，这样的话当宿主机的用户为`root`时就有权限写mount的目录。
3. 当通过docker-compose编排好之后，通过'http://ip:3000'登录grafana，并且配置好prometheus数据源，再将dashboard模板https://github.com/Vertamedia/chproxy/blob/master/chproxy_overview.json导入之后发现没有数据，连左上角的job下拉框都没有任何数据。然后我通过当前dashboard的`Variables`发现下拉框job的内容是通过prometheus中`go_info`来获取的，但是我通过promql查询`go_info`中没有`prometheus.yml`中配置的`-job_name`为`clickhouse-chproxy`的内容。但是我发现`go_goroutines`中有，于是我将`go_info`替换成了`go_goroutines`。
4. 经过测试返现只有CHproxy至少经过一次的使用查询之后`go_goroutines`中才有`-job_name`为`clickhouse-chproxy`的内容，其他metrics也是才会出现。



go_goroutines

go_info没有别的信息，需要使用接口发送一次查询

grafana mount 需要 root user 当前host 为root

### 题外话

我直接在grafana容器中安装了clickhouse DataSource插件，并且制作成了镜像，这样的话grafana也可以直接查询clickhouse了哦，参考：https://github.com/Vertamedia/clickhouse-grafana，https://grafana.com/plugins/vertamedia-clickhouse-datasource/installation。