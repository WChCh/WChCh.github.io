---
title: Java应用问题排查笔记
date: 2017-11-07 17:03:00
tags: [Java虚拟机,JVM,问题排查,OutOfMemoryError]
categories: [技术,学习,JVM]
---

前一段时间跑在tomcat的java应用出现线程挂掉，CPU占用率持续过高，堆溢出的问题。首先说一下这件事的经过。

我在我的模块中有两个定时任务，使用的是spring boot的注解`@Scheduled(fixedDelay=ONE_Minute)`，后来出现这了两个定时任务都不再往redis缓存里面存放数据的问题。一开始没有经验，也没有打印足够的日志，所以从日志里面也看不出啥问题。

后来，我去除掉了定时任务，改成了直接new线程的方式（这种方式当然不好），并且详细打上了日志。但又出现问题了，但可喜的是这两个线程中只有一个不工作了（就是不缓存数据了，具体问题也不清楚），另外一个还很正常。这种情况出现了2到3次，我这时候还是束手无策，不知为何？

<!--more-->

我继续在网上搜索答案，说是用`jps`,`jstat`,`jinfo`,`jmap`,`jhat`,`jstack`这些命令可以做故障分析。下面是使用的过程。

首相使用`jps`查询运行tomcat的JVM虚拟机的进程：

```
jps
```

得到一下结果：

```
2667 Bootstrap
3116 Jps
```

其中`Bootstrap`代表的`2667`就是tomcat java虚拟机的进程号。有时候使用`jps`命令无法得到类似`2667 Bootstrap`的结果(好像重启tomcat之后好使)，那么可用`ps -ef | grep tomcat`得到进程号。

#### 判断线程状态

首先通过`jstack -l pid`来查看所有的线程栈,若果调用失败可使用`stack -F pid`强制导出。通过导出的线程堆栈信息来判断线程的状态，是否还存在？如果存在的话，处于什么状态？

#### 判断Java进程为何CPU占用率高

可通过`ps -Lfp pid`或者`ps -mp pid -o THREAD, tid, time`或者`top -Hp pid`来判断java进程中哪个线程的CPU占用过高。

确定了线程的id之后，再通过`jstack -l pid`打印出的信息查看对应的线程的堆栈，确认线程的工作状态。参考链接：<https://my.oschina.net/feichexia/blog/196575>

#### 判断OutOfMemoryError问题

当发生`OutOfMemoryError`时希望把当时的内存数据保存下来以便于使用可视化工具（eclipse memory analyzer ）分析，可以在tomcat bin下的catalina.sh文件中加入相关参数。添加位置如下：

```
fi

# ----- Execute The Requested Command -----------------------------------------

JAVA_OPTS="$JAVA_OPTS -Xms1024m -Xmx1024m  -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/cloudview/data/dump/oom.hprof"

# Bugzilla 37848: only output this if we have a TTY
if [ $have_tty -eq 1 ]; then
```

`-XX:HeapDumpPath=/cloudview/data/dump/oom.hprof`表示文件保存的路径以格式。

发生`OutOfMemoryError`可能会导致线程异常终止，可用`catch Throwable`捕捉并打印最后的堆栈信息。另外不建议单独使用new线程的方式，可使用线程池来保证重启线程。