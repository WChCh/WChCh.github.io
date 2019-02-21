---
layout: spring
title: 外置application.properties和logback.xml配置文件
date: 2019-02-21 14:53:47
tags: [spring boot,java,配置]
categories: [spring boot,java]
---

#### 自定义打包输出

使用`maven-assembly-plugin`自定义打包输出，其在pom文件中的配置如下：

```
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-assembly-plugin</artifactId>
    <configuration>
        <finalName>4a-insight-clickhouse</finalName>
        <descriptors>
            <descriptor>src/main/assembly/package.xml</descriptor>
        </descriptors>
    </configuration>
    <executions>
        <execution>
            <!-- 绑定到package生命周期阶段上 -->
            <phase>package</phase>
            <goals>
                <!-- 只运行一次 -->
                <goal>single</goal>
            </goals>
        </execution>
    </executions>
    <resources>
    <resource>
        <directory>src/main/resources</directory>
        <excludes>
            <exclude>logback-spring.xml</exclude>
        </excludes>
    </resource>
</resources>
</plugin>
```

`<descriptors>`描述自定义文件的位置，`<phase>`maven打包的哪个生命周期生效。由于有了外置的`logback-spring.xml`文件，所以要排除掉jar包内的`logback-spring.xml`文件，要不然会出现错误。

<!--more-->

`package.xml`配置文件：

```xml
<assembly
        xmlns="http://maven.apache.org/plugins/maven-assembly-plugin/assembly/1.1.0"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://maven.apache.org/plugins/maven-assembly-plugin/assembly/1.1.0 http://maven.apache.org/xsd/assembly-1.1.0.xsd">
    <id>full</id>
    <formats>
        <format>zip</format>
    </formats>
    <includeBaseDirectory>false</includeBaseDirectory>
    <fileSets>
        <!-- for bin -->
        <fileSet>
            <directory>src/main/bin</directory>
            <includes>
                <include>*.*</include>
            </includes>
            <directoryMode>775</directoryMode>
            <outputDirectory>/bin</outputDirectory>
        </fileSet>
        <!-- for spring boot jar -->
        <fileSet>
            <directory>target/</directory>
            <includes>
                <include>9n-4a-insight-0.0.1-SNAPSHOT.jar</include>
            </includes>
            <outputDirectory>/lib</outputDirectory>
        </fileSet>

        <fileSet>
            <!--打包时把配置文件放在config目录下，并且config目录与jar包同属于一个目录-->
            <directory>target/classes/</directory>
            <includes>
                <include>application.properties</include>
                <include>logback-spring.xml</include>
            </includes>
            <outputDirectory>/lib/config</outputDirectory>
            <fileMode>0755</fileMode>
        </fileSet>
    </fileSets>
</assembly>

```

将`jar`包放到`lib`目录下，`application.properties`和`logback-spring.xml`放到`lib/config`目录下，启动和停止脚本放到`bin`目录下。关于`application.properties`配置文件生效优先级参考：[https://www.jianshu.com/p/86a40ce2eb7a](https://www.jianshu.com/p/86a40ce2eb7a)，logback扩展配置参考：[https://juejin.im/post/5bbdb64fe51d450e5e0cb1c4](https://juejin.im/post/5bbdb64fe51d450e5e0cb1c4)。

#### 应用启动命令配置

根据命令执行的***当前路径***找到对应的配置文件

```shell
#!/usr/bin/env bash
java -jar  -Dspring.config.location=../lib/config/ -Dlogback.configurationFile=../lib/config/logback-spring.xml ../lib/9n-4a-insight-0.0.1-SNAPSHOT.jar &
```