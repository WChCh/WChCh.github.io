---
title: Clickhouse 使用总结
date: 2018-12-20 11:58:50
tags: [clickhosue,olap,MPP]
categories: [olap,BigData,clickhouse,大数据]
---

### Clickhouse 使用总结

以下是本人短时间内的clickhouse调研使用总结，很多坑还没踩过，所以只是一些浅显的介绍。

#### 配置文件

clickhouse的配置文件包括`/etc/clickhouse-server/`下的`config.xml`，`users.xml`以及`/etc/`下的集群配置文件`metrika.xml`。通过`metrika.xml`可以看到节点登陆的明文用户名，密码。关于配置文件的说明不再赘述。

#### 客户端

clickhouse提供连接的客户端有好几种，有命令行客户端，JDBC驱动的客户端和HTTP客户端等，这里主要将如何在spring boot中封装HTTP客户端和基于JDBC驱动集成Mybatis。

##### JDBC

JDBC驱动使用官方提供的，走8123端口，使用http协议。官方驱动连接地址为：[https://github.com/yandex/clickhouse-jdbc](https://github.com/yandex/clickhouse-jdbc)

<!--more-->

首先`POM`文件中加入依赖：

```java
<dependency>
	<groupId>org.mybatis.spring.boot</groupId>
	<artifactId>mybatis-spring-boot-starter</artifactId>
	<version>1.1.1</version>
</dependency>
<dependency>
    <groupId>ru.yandex.clickhouse</groupId>
    <artifactId>clickhouse-jdbc</artifactId>
    <version>0.1.43</version>
</dependency>
```

然后基于注解编写Mybatis的Mapper接口：

```java
@Mapper
public interface UserMapper {

    @Select("SELECT * FROM user WHERE name = #{name} FORMAT TabSeparatedWithNamesAndTypes")
    User findByName(@Param("name") String name);

    @Insert("INSERT INTO user(id, name, age,date) VALUES(#{id}, #{name}, #{age}, #{date})")
    int insert(@Param("id") Long id, @Param("name") String name, @Param("age") Integer age, @Param("date") String date);
}
```

**注意**：官方提供的JDBC驱动提供的查询`FORMAR`方式只支持`TabSeparatedWithNamesAndTypes`方式，经测试，在版本`0.1.43`中当查询语句有`where` 等条件时要加上`FORMAT TabSeparatedWithNamesAndTypes`。

配置文件`application.properties`的配置如下：

```java
# clickhouse
spring.datasource.url=jdbc:clickhouse://localhost:8123/default
spring.datasource.driver-class-name=ru.yandex.clickhouse.ClickHouseDriver
spring.datasource.username=root
spring.datasource.password=123456
```

##### HTTTP

HTTP客户端同样是走8123端口，网上有开源的基于Java 8封装的客户端，但是由于公司环境还只是Java 7，所以我基于`httpclient`自己封装了一个。

首先添加依赖：

```java
<dependency>
	<groupId>org.apache.httpcomponents</groupId>
	<artifactId>httpcore</artifactId>
	<version>4.4.10</version>
</dependency>

<dependency>
	<groupId>org.apache.httpcomponents</groupId>
	<artifactId>httpclient</artifactId>
	<version>4.5.6</version>
</dependency>

<dependency>
	<groupId>com.alibaba</groupId>
	<artifactId>fastjson</artifactId>
	<version>1.2.15  </version>
</dependency>
```

这里引入Json依赖主要是用于接收Json格式的数据。

然后创建`clickhouseHttpClient`类：

```java
import java.io.IOException;
import java.net.URISyntaxException;
import com.alibaba.fastjson.JSONObject;
import org.apache.commons.codec.binary.Base64;
import org.apache.http.HttpEntity;
import org.apache.http.HttpResponse;
import org.apache.http.client.HttpClient;
import org.apache.http.client.methods.HttpGet;
import org.apache.http.client.utils.URIBuilder;
import org.apache.http.impl.client.HttpClientBuilder;
import org.apache.http.util.EntityUtils;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;


public class ClickHouseClient implements AutoCloseable {
	private static final Logger LOG = LoggerFactory.getLogger(ClickHouseClient.class);

	private static final String SELECT_FORMAT = "JSON";
	private static final String INSERT_FORMAT = "TabSeparated";

	private final String endpoint;
	private final HttpClient httpClient;
	private final HttpGet httpGet;
	private final URIBuilder builder;
	private HttpEntity httpEntity;

	public ClickHouseClient(String endpoint, String username, String password) throws URISyntaxException{
		this.endpoint = endpoint;
		this.builder = new URIBuilder(endpoint);
		this.httpGet = new HttpGet();
		String encoding = Base64.encodeBase64String((username + ":" + password).getBytes());
		this.httpGet.setHeader("Content-type", "application/json");
		this.httpGet.setHeader("charset", "UTF-8");
		this.httpGet.setHeader("Authorization", "Basic " + encoding);
		this.httpClient = HttpClientBuilder.create().build();
	}

	public void close() {
		try {
		    if(null != this.httpEntity){
                EntityUtils.consume(this.httpEntity);
            }
		} catch (Exception e) {
			LOG.error("Error closing http client", e);
		}
	}

	public JSONObject get(String query) throws URISyntaxException, IOException {
		String queryWithFormat = query + " FORMAT " + SELECT_FORMAT;

		this.builder.setParameter("query", queryWithFormat);
		this.httpGet.setURI(this.builder.build());
		
		LOG.debug("querying GET {}", queryWithFormat);

		HttpResponse response = this.httpClient.execute(httpGet);
		this.httpEntity = response.getEntity();
		JSONObject json = JSONObject.parseObject(EntityUtils.toString(this.httpEntity));
		System.out.println(json);
		return json;
	}
}

```

使用HTTP客户端的好处是，当查询语句中进行`group by`后，紧跟着`WITH TOTALS`，并且是`FORMAT JSON`时能够返回总数，这使得可以计算百分比。

使用示例：

```java
  public static void main(String[] args) throws Exception{
        ClickHouseClient clickHouseClient = new ClickHouseClient("http://11.40.243.166:8123", "root", "123456");
        String table = " from default.test_table ";
        String where = " where status = '" + status + "' and dt ='" + dt + "' and brand_id = '" + brand_id + "' and item_third_cate_id= '" + item_third_cate_id + "' ";
        String totals = " WITH TOTALS ";
        String format = "";
        String exp = " count(*) as value ";
        String sex = "select cpp_base_sex as name, " + exp + table + where + " group by cpp_base_sex " + totals + format;
        JSONObject sexJson = clickHouseClient.get(sex);
    }
```



#### 建表

目前的集群环境中只做了三个分片没有做副本，所以我使用的是`MergeTree`引擎。建表过程包括建本地表和分布式表。

##### 本地表

```sql
CREATE TABLE IF NOT EXISTS table_test_local
(
    brand_id String, 
    user_log_acct String, 
    status String, 
    dt String
)
ENGINE = MergeTree
PARTITION BY (dt, status)
ORDER BY (brand_id)
```

`PARTITION BY`表示创建分区，`ORDER BY (brand_id)`表示索引，可以创建联合索引，并且遵守左生效原则，我这里没有指定索引粒度，使用的默认值。

##### 分布式表

```sql
CREATE TABLE table_test
(
    brand_id String,
    user String,
    status String, 
    dt String
)ENGINE = Distributed(jdos_clickhouse_cluster, insight, 'table_test_local', rand());
```

`jdos_clickhouse_cluster`表示某一个集群，可在配置文件中找到，`insight`表示某一个库，`table_test_local`表示关联的本地表，rand()表示如何将数据分发到某一个分片上，这里采用随机的方式。

*注意*：导入输入时clickhouse默认将null转为了长度为零的空字符串。

#### SQL与优化

clickhouse提供了很好的SQL查询支持，很容易上手，下面主要介绍我用到的一些高阶SQL以及优化方法。

##### 高阶SQL

`select if(empty(dim_jxkh_level), '-1', dim_jxkh_level) as name`表示`dim_jxkh_level`为空时返回`-1`

`select arrayJoin(arrayFilter(element -> notEmpty(element), splitByString('#',cfv_cate_30dcate3))) as name`:

1. `splitByString('#',cfv_cate_30dcate3)`表示字符串按住奥`#`切分，并且返回数组。
2. `arrayFilter(element -> notEmpty(element), splitByString('#',cfv_cate_30dcate3))`通过lamda表达式`element -> notEmpty(element)`过滤掉数组`splitByString('#',cfv_cate_30dcate3)`中为空的字符。
3. `arrayJoin()`表示按照某一列中的数组内容进行数据展开成多列，展开的这些列不同的地方只是原先数据所在的列，与spark中的`explode`方法功能一致。
4. 可使用字典替代join加速查询。字典的介绍：[https://www.altinity.com/blog/2017/4/12/dictionaries-explained](https://www.altinity.com/blog/2017/4/12/dictionaries-explained)。

#### 优化

主要是本人在这一两周内的使用心得，涉及到的优化层面还是很小，以后如果有新的东西再添加。

##### 集群优化

使用集群分片的分布式查询方式在大规模数据下会有显著的效果。

##### 建表优化

建表时请指定分区和主键索引，索引遵循左优先原则。索引表将会贮存在内存中，xxx.mrk文件也将会被缓存。MergeTree的工作原理以及索引查找原理参考如下链接：

1. [https://clickhouse.yandex/docs/en/development/architecture/#merge-tree](https://clickhouse.yandex/docs/en/development/architecture/#merge-tree)
2. [http://www.clickhouse.com.cn/topic/5a366e48828d76d75ab5d59e](http://www.clickhouse.com.cn/topic/5a366e48828d76d75ab5d59e)

##### SQL优化

SQL优化是一个很重要的手段，用好了能达到很高的优化性能。

1. 尽量不适用子查询。

2. 如果count(*)能够达到同等的效果，就不要使用count()某一列。

3. 使用`union all`时，会出现所union的多条sql的并发执行，如果cpu资源不足会很耗时，具体方法请看并发优化这一小节。

4. 分组并且分组内取`topN`时，可以使用`limit N by colunm`的方式：

   ```sql
   # 以brand_id做分组，取每个组内top N 的name
   select brand_id, name, count(*) as value from table group by brand_id, name order by value limit N by brand_id limit 100;
   ```

5. 再某些情况下使用`prewhere`作用于非索引列进行优化，`prewhere`的使用介绍：[prewhere 作用于非索引列 https://github.com/yandex/ClickHouse/issues/2601](https://github.com/yandex/ClickHouse/issues/2601)

##### 并发优化

Clickhouse支持并发，并且支持的最大并发数能够在配置文件里面配置，默认为一百。但是clickhouse的并发性能不好，经过测试还倒不如串行执行的效果好。本人的环境是8核心，32G内存的3台主机3分片的集群，执行单条SQL时通过`top`命令查看，3台机器的8个核心都跑满，我想这也是MPP架构以及clickhouse的厉害之处，能够充分利用计算资源，所以我认为当并发执行的时候会导致CPU并发的上下文切换花费过多时间。

既然每条SQL都能够将CPU跑满，那么是否可以控制单个查询cpu的使用个数。通过查文档，可以通过`max_threads`来设置，而且`max_threads`的默认数就刚好为8。我将`max_threads`设置为4，然后并发执行两条SQL，将结果与`max_threads`设置为8串行执行两条sql进行对比，时间性能上有一点点提高，虽然不明显，但可喜。

除了设置`max_threads`，考虑到集群查询做汇总时主节点的计算压力比较大，所以我再所有的3个分片节点上都建了分布式表，将所有sql平均分给3个主机，每个主机的并发度为1，`max_threads`设置为2，理想状况下就不会有CPU竞争情况。

#### 系统库system

通过`system`库里的各种表可以监控clickhouse的性能以及各种配置情况，比如通过`log_queries`表可以查询各种sql的执行情况，但是此表默认不开启。