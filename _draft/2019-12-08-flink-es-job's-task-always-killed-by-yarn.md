---
layout: post
title:  "一个最简单的Flink Job，一次最复杂的问题排查"
categories: flink
tags:  flink elasticsearch yarn
author: tiny
---

* content
{:toc}


## 概括

本文主要是记录一个简单的`Flink Job`在从Standalone迁移OnYarn后所遇到的一个因内存占用超出限制而引发的Container频繁被Yarn Kill的问题。该问题的排查解决过程主要经历了：Flink监控指标分析，GC日志的排查，TaskManger内存分析，Container的内存计算方法，栈内存的分析等内容；

## 问题描述

**起因：** 希望将`Flink Standalone`上一个简单的Job迁移到`Flink On Yarn`，迁移前的版本为`Flink 1.3.2`，迁移目标版本为`Flink 1.7.1 + Hadoop 3.1.0`；

**Job描述：**

- 有10个`Kafka Topic`，每个Topic的partition数为21；
- 每个`Kafka Topic`对应一个`ES Index`；
- 读取所有Topic的数据，筛选出包含指定字段的数据并将结果写入各Topic对应的`Es Index`；
- 任务提交脚本：`flink run -m yarn-cluster -yn 11 -ys 2 -ytm 6g -ynm original-data-filter-prod -d -yq XXX.jar 21 app prod`，即：给Job分配11个Container，每个Container[2C4G]持有2个Slot，任务并行度为21；Job逻辑视图如下：

```mermaid
graph LR
A(Source: KafkaSource) --> B(Operator: Filter)
B --> C(Sink: Es Index)
```
`KafkaSource`: 10个Topic * 21个Parallelism；
`Filter`: 与KafkaSource保持一致；
`Es Sink`: 10个Index * 21个Parallelism；

**异常描述：**

Job正常启动后TaskManager所在的Container每3～5min会被Yarn Kill掉，然后Job的AppMaster会重新向ResourceManager申请一个新的Container以启动之前被Kill掉的Container里的Task，整个Job会陷入不间断的`Kill Container`/`Apply For New Container`/`Start New Task`的循环，在Flink和Yarn的日志里都会发现如下错误信息：

```java
2019-11-25 20:12:41,138 INFO  org.apache.flink.yarn.YarnResourceManager - Closing TaskExecutor connection container_e03_1559725928417_0577_01_001065 because: [2019-11-25 20:12:36.159]Container [pid=97191,containerID=container_e03_1559725928417_0577_01_001065] is running 168853504B beyond the 'PHYSICAL' memory limit. Current usage: 6.2 GB of 6 GB physical memory used; 9.8 GB of 60 GB virtual memory used. Killing container.
Dump of the process-tree for container_e03_1559725928417_0577_01_001065 :
        |- PID PPID PGRPID SESSID CMD_NAME USER_MODE_TIME(MILLIS) SYSTEM_TIME(MILLIS) VMEM_USAGE(BYTES) RSSMEM_USAGE(PAGES) FULL_CMD_LINE
        |- 97215 97191 97191 97191 (java) 87864 3747 10359721984 1613717 /usr/java/default/bin/java -Xms4425m -Xmx4425m -XX:MaxDirectMemorySize=1719m -XX:+UseG1GC -XX:MaxGCPauseMillis=100 -XX:+ParallelRefProcEnabled -XX:ErrorFile=/home/hadoop/apache/hadoop/latest/logs/userlogs/application_1559725928417_0577/container_e03_1559725928417_0577_01_001065/hs_err_pid%p.log -Xloggc:/home/hadoop/apache/hadoop/latest/logs/userlogs/application_1559725928417_0577/container_e03_1559725928417_0577_01_001065/gc.log -XX:HeapDumpPath=/home/hadoop/apache/hadoop/latest/logs/userlogs/application_1559725928417_0577/container_e03_1559725928417_0577_01_001065 -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+HeapDumpOnOutOfMemoryError -Dlog.file=/home/hadoop/apache/hadoop/latest/logs/userlogs/application_1559725928417_0577/container_e03_1559725928417_0577_01_001065/taskmanager.log -Dlogback.configurationFile=file:./logback.xml -Dlog4j.configuration=file:./log4j.properties org.apache.flink.yarn.YarnTaskExecutorRunner --configDir .
        |- 97191 97189 97191 97191 (bash) 0 0 118067200 371 /bin/bash -c /usr/java/default/bin/java -Xms4425m -Xmx4425m -XX:MaxDirectMemorySize=1719m -XX:+UseG1GC -XX:MaxGCPauseMillis=100 -XX:+ParallelRefProcEnabled -XX:ErrorFile=/home/hadoop/apache/hadoop/latest/logs/userlogs/application_1559725928417_0577/container_e03_1559725928417_0577_01_001065/hs_err_pid%p.log -Xloggc:/home/hadoop/apache/hadoop/latest/logs/userlogs/application_1559725928417_0577/container_e03_1559725928417_0577_01_001065/gc.log -XX:HeapDumpPath=/home/hadoop/apache/hadoop/latest/logs/userlogs/application_1559725928417_0577/container_e03_1559725928417_0577_01_001065 -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+HeapDumpOnOutOfMemoryError -Dlog.file=/home/hadoop/apache/hadoop/latest/logs/userlogs/application_1559725928417_0577/container_e03_1559725928417_0577_01_001065/taskmanager.log -Dlogback.configurationFile=file:./logback.xml -Dlog4j.configuration=file:./log4j.properties org.apache.flink.yarn.YarnTaskExecutorRunner --configDir . 1> /home/hadoop/apache/hadoop/latest/logs/userlogs/application_1559725928417_0577/container_e03_1559725928417_0577_01_001065/taskmanager.out 2> /home/hadoop/apache/hadoop/latest/logs/userlogs/application_1559725928417_0577/container_e03_1559725928417_0577_01_001065/taskmanager.err

[2019-11-25 20:12:36.175]Container killed on request. Exit code is 143
[2019-11-25 20:12:38.293]Container exited with a non-zero exit code 143.

```




## 问题分析


## 总结
在起初对Job的思考中是想尽量减少资源占用，最极端的办法就是将数据读取/处理/写出的整个流程放在同一个JVM中，这样Operator间的数据传递就可以在JVM内部完成而非需要在网络间传递数据；由此，进一步想到`Chain All Operators Together`，这样就必须把所有`source/filter/sink`的并行度设置为相同的值；然而，这就造成每个Container中的Operator数量增加，特别是`Es Sink`是个较为重的Operator（内部维护的线程/缓存/状态较多）；`Es Sink`实例对象的增加导致了线程数的成倍增加，所有线程持有的Stack Memory总和也成倍增加，用于运行Job Task的Memory数量进一步减少，另外一个Container所持有的2C的CPU资源其实也很难支撑这么多的线程数正常运行。

所以，在了解到上述内容后，首先需要做的事情是减少Container中的线程数（降低Stack Memory占用，减少线程切换），减少线程办法就是减少`ES Sink Operator`的数量（source/filter线程少可忽略不计），从得出了当前的解决方案；

最后，将`Es Sink Operator`并行度调整为2后重启Job，问题得到彻底解决，至今Job已平稳运行了10多天；

## Reference


cGroup: /proc/<pid>/stat
proc: https://www.cnblogs.com/yurunmiao/p/5070287.html
yarn.nodemanager.container-monitor.process-tree.class

container_e03_1559725928417_0572_01_000002
less hadoop-hadoop-nodemanager-bj-xg-app-flink-007.tendcloud.com.log


首先判断堆内外oom
堆外占用也增长很快
定义Xmx等后不是应该报oom，为什么会被kill
Xss 默认 1M，1000+个线程
java stack详解，与堆的关系，与-Xmx的关系
container内存组成：taskmanager.heap.size=4g(heap, noff-heap, non-heap, stack)
列举一个task线程，将task线程中所有对象split到各个区域heap，off-heap，stack等等；
关于stack的详细描述，FIFO，堆外，有各自独立的计数器 。。。
将container的内存6g分割开 堆4G，堆外4G，Flink container组成，managed、.....
jcmd 36337 VM.native_memory summary scale=MB
-ea -Xmx100m -Xms100m -Xss5m -XX:MaxDirectMemorySize=10m -XX:NativeMemoryTracking=detail

jobmnager无限次向yarn申请新的container问题解决，如何快速失败；

=====expand thinking=====
任务失败有多种，yarn kill container， job internal logic occur exception
==========
flink preallocate memory
flink与spark比多出了slot share 这种概念，同时也因为是pipline很容易在内存，cpu，等方面出问题

写代码时候的编程模式：鱼骨（复杂业务逻辑） vs 瀑布（数据处理）
