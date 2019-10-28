---
title: 制作可传参的docker clickhouse-client
date: 2019-10-27 20:26:51
tags: [clickhosue,olap,MPP]
categories: [olap,BigData,clickhouse,大数据]
---

Dockerfile：

```shell
FROM clickhouse-client:latest

ENV HOST="localhost"
ENV USER="default"
ENV PASSWORD=""
ENV DATABASE="default"
ENTRYPOINT /usr/bin/clickhouse-client -h $HOST -u $USER --password $PASSWORD -d $DATABASE
```

运行命令：

```shell
docker run -it --rm --name clickhouse-client -e HOST="10.185.xx.xx" -e USER="user" -e PASSWORD="xxxx" -e DATABASE="db"  clickhouse-client:param
```

遇到问题：

当ENTRYPOINT使用方括号并且里面的参数是明文而不是环境变量时，是可以运行的，如

```
ENTRYPOINT ["/usr/bin/clickhouse-client","-h","10.185.xx.xx","-u","user","--password","xxx"]
```

但是形如：

```shell
ENTRYPOINT ["/usr/bin/clickhouse-client","-h",$HOST,"-u",$USER,"--password",$PASSWORD]
```

就会报：

```shell
/bin/sh 1 [/usr/bin/clickhouse-client,-h,10.185.xx.xx,-u,user,--password,password] not found
```

