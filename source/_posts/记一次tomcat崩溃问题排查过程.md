---
title: 记一次tomcat崩溃问题排查过程
date: 2017-11-10 17:05:27
tags: [Java虚拟机,tomcat,JVM,问题排查,OutOfMemoryError]
categories: [技术,学习,JVM]
---

产品在生产环境中跑了一段时间后，每天早晨运维人员都会发现程序崩溃，无法提供服务，重启之后就好了，而且白天都没问题，运维人员因此叫苦连天。

查看日志发现有堆内存溢出的提示，并且都是发生在凌晨一点钟左右，但找不到是哪个线程导致的，重启之后通过jstack、jmap、jstat等命令查看各项指标都正常。无奈只能在`catalina.sh`配置文件中加入`JAVA_OPTS="$JAVA_OPTS -Xms1024m -Xmx1024m -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/.../oom.hprof"`配置项，当程序因为内存溢出崩溃时能够查看当时的jvm内存使用情况。

<!--more-->

隔天早上再次发生崩溃，产生了`oom.hprof`文件，让运维人员传过来，导入`MemoryAnalyzer`中进行分析。分析过程中发现某个定时任务的线程对象占用了将近一半的堆内存，并且对象类型为：

```
java.lang.Object[4102267] @ 0xe817b3f0
 16,409,088 864,074,952 48.56% 
..+com.mysql.jdbc.ByteArrayRow @ 0xb0176c20
 24 304 0.00% 
..+com.mysql.jdbc.ByteArrayRow @ 0xe3044560
 24 304 0.00% 
..+com.mysql.jdbc.ByteArrayRow @ 0xce6ac738
 24 304 0.00% 
..+com.mysql.jdbc.ByteArrayRow @ 0xaf2ab198
 24 304 0.00% 
..+com.mysql.jdbc.ByteArrayRow @ 0xa82fbcc0
 24 304 0.00% 
..+com.mysql.jdbc.ByteArrayRow @ 0xcbce65f8
 24 304 0.00% 
..+com.mysql.jdbc.ByteArrayRow @ 0xb1b9e200
 24 304 0.00% 
..+com.mysql.jdbc.ByteArrayRow @ 0xb515f4a0
 24 304 0.00% 
..+com.mysql.jdbc.ByteArrayRow @ 0xde9fab68
 24 304 0.00% 
..+com.mysql.jdbc.ByteArrayRow @ 0xac3662a8
 24 304 0.00% 
..+com.mysql.jdbc.ByteArrayRow @ 0xd3b9e6f8
 24 304 0.00% 
..+com.mysql.jdbc.ByteArrayRow @ 0xe3044690
 24 304 0.00% 
..+com.mysql.jdbc.ByteArrayRow @ 0xa51149a8
 24 304 0.00% 
..+com.mysql.jdbc.ByteArrayRow @ 0xafc57d70
 24 304 0.00% 
..+com.mysql.jdbc.ByteArrayRow @ 0x8f536ab0
 24 304 0.00% 
..+com.mysql.jdbc.ByteArrayRow @ 0xd6bb6588
 24 304 0.00% 
..+com.mysql.jdbc.ByteArrayRow @ 0xb0d39428
 24 304 0.00% 
..+com.mysql.jdbc.ByteArrayRow @ 0x98c5da88
 24 304 0.00% 
..+com.mysql.jdbc.ByteArrayRow @ 0xb2de5950
 24 304 0.00% 
..+com.mysql.jdbc.ByteArrayRow @ 0xde9fac98
```

通过在网上搜索答案，初步猜测是一次从数据库中读取了大量的数据到内存中导致内存溢出的崩溃问题。接着查看线程栈，定位到了线程中导致问题的某一处代码。

这一处代码做的是一个根据条件定期删除数据库的操作，这是经过本地调试发现每次做删除操作时并没有报异常，但也没删除成功。此时可以了解了问题发生的原因，由于此每次都没能成功删除数据，但是还会有新数据不断加入，日积月累数据量会越来越庞大。每次做删除操作时JPA预先加载所有要删除的数据到内存中，这时大数据量会导致JVM实例堆溢出崩溃。

定位到问题代码之后要分析一下为什么没有成功删除数据。这里要先说一下，我们项目中使用的是spring boot，持久层api使用的是spring data jpa。通过分析发现代码中没有在service层的实现类或是相关方法上加`@transactional`，导致的删除数据时事务不生效。

但是当我给service层实现类加上`@transactional`时，删除数据仍然不成功，分析代码发现，他这个service层实现类不但有implements接口，还extends抽象类，并不是纯碎的接口实现，即使直接加在方法上也不行，为什么不成功我暂时还没有时间去研究。无奈只能在repository层加，操作才成功，显然这种方法很不好！没办法，先交给代码关系人来解决吧，我也没时间和精力！