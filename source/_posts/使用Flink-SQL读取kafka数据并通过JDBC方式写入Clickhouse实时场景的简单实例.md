---
title: 使用Flink SQL读取kafka数据并通过JDBC方式写入Clickhouse实时场景的简单实例
date: 2019-11-27 16:43:55
tags: [clickhosue,olap,MPP,并发,代理,flink]
categories: [实时,olap,BigData,clickhouse,大数据]
---

### 说明

读取kafka数据并且经过ETL后，通过JDBC存入clickhouse中

### 代码

POM文件：

```xml
       <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-java</artifactId>
            <version>${flink.version}</version>
            <!--<scope>provided</scope>-->
        </dependency>
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-streaming-java_${scala.binary.version}</artifactId>
            <version>${flink.version}</version>
           <!-- <scope>provided</scope>-->
        </dependency>

        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-streaming-scala_${scala.binary.version}</artifactId>
            <version>${flink.version}</version>
            <!-- <scope>provided</scope>-->
        </dependency>          
        <dependency>
            <groupId>ru.yandex.clickhouse</groupId>
            <artifactId>clickhouse-jdbc</artifactId>
            <version>0.2</version>
        </dependency>

        <dependency>
            <groupId>org.apache.httpcomponents</groupId>
            <artifactId>httpcore</artifactId>
            <version>4.4.4</version>
        </dependency>

        <dependency>
            <groupId>com.google.guava</groupId>
            <artifactId>guava</artifactId>
            <version>19.0</version>
        </dependency>

        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-jdbc_${scala.binary.version}</artifactId>
            <version>${flink.version}</version>
        </dependency>
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-json</artifactId>
            <version>${flink.version}</version>
        </dependency>

        <!-- Either... -->
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-table-api-java-bridge_${scala.binary.version}</artifactId>
            <version>${flink.version}</version>
        </dependency>

        <!-- Add connector dependencies here. They must be in the default scope
            (compile). -->
        <!-- this is for kafka consuming -->
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-connector-kafka-0.10_${scala.binary.version}</artifactId>
            <version>${flink.version}</version>
        </dependency>
       <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-table-planner_${scala.binary.version}</artifactId>
            <version>${flink.version}</version>
       </dependency>
```

<!--more-->

定义POJO类：

```java
public class Student {
    private int id;
    private String name;
    private String password;
    private int age;
    private String date;
    //构造，setter 和 getter 省略
}
```

完整代码：

```java
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
env.setParallelism(1);
final StreamTableEnvironment tableEnv = StreamTableEnvironment.create(env);

//###############定义消费kafka source##############
Properties props = new Properties();
props.put("bootstrap.servers", "localhost:9092");
props.put("zookeeper.connect", "localhost:2181");
props.put("group.id", "metric-group");
props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
props.put("value.deserializer","org.apache.kafka.common.serialization.StringDeserializer");
props.put("auto.offset.reset", "latest");

tableEnv.connect(new Kafka().version("0.10")
        .topic("student").properties(props).startFromLatest())
        .withFormat(new Json().deriveSchema())
        .withSchema(new Schema().field("id", Types.INT())
                                .field("name", Types.STRING())
                                .field("password", Types.STRING())
                                .field("age", Types.INT())
                                .field("date", Types.STRING()))
        .inAppendMode()
        .registerTableSource("kafkaTable");
Table result = tableEnv.sqlQuery("SELECT * FROM " +  "kafkaTable");

//###############定义clickhouse JDBC sink##############
String targetTable = "clickhouse";
TypeInformation[] fieldTypes = {BasicTypeInfo.INT_TYPE_INFO,BasicTypeInfo.STRING_TYPE_INFO,BasicTypeInfo.STRING_TYPE_INFO, BasicTypeInfo.INT_TYPE_INFO, BasicTypeInfo.STRING_TYPE_INFO};
TableSink jdbcSink =  JDBCAppendTableSink.builder()
                      .setDrivername("ru.yandex.clickhouse.ClickHouseDriver")
                      .setDBUrl("jdbc:clickhouse://localhost:8123")
                      .setQuery("insert into student_local(id, name, password, age, date) values(?, ?, ?, ?, ?)")
                      .setParameterTypes(fieldTypes)
                      .setBatchSize(15)
                      .build();

tableEnv.registerTableSink(targetTable,new String[]{"id","name", "password", "age", "date"}, new TypeInformation[]{Types.INT(), Types.STRING(), Types.STRING(), Types.INT(), Types.STRING()}, jdbcSink);

result.insertInto(targetTable);
env.execute("Flink add sink");


```

