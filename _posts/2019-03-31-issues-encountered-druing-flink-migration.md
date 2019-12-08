---
layout: post
title:  "Flink Standalone/OnYarn使用过程中遇到的若干问题记录"
categories: flink
tags:  flink yarn
author: tiny
---

* content
{:toc}


## 前言

`Flink`是由公司前辈引入，最初版本是`Flink 1.1.5`，后升级到`Flink 1.3.2`一直以来`Flink Job`都是跑在`Standalone`集群上的，为规避`Standalone`集群中`Job`相互影响的问题（`Standalone`无法做资源隔离），前辈们又新做了`Flink`集群，将互相影响的`Job`拆分到不同集群中，这时团队内部有多个`Flink Standalone`集群，虽然解决了部分`Job`互相影响的问题，但有些服务是和`Kafka`，`Druid`部署在同一台物理机器的（不同进程），又出现过几次不同服务互相影响的情况；这些`Flink Standalone`集群维护/升级起来也挺麻烦的，同时新`Job`越来越多，生活苦不堪言~.~；兄弟们从2018年初就喊着`OnYarn`，结果由于人力、机器、业务迭代等各种问题一拖再拖；到2018年底终于开始了这项工作，虽然只1个人力投入，经历2-3个月的时间团队的`streaming Job`已经稳定运行在`OnYarn`模式下，本文就重点记录下这段时所遇到的一些重点问题，当前生产运行版本是`Flink 1.7.1 + Hadoop 3.1.0`

## 问题汇总

### Flink与Hadoop版本兼容性问题

由于官方没有依赖`Hadoop 3.1.0`的`Flink`下载包，所以这步需要自行编译`Flink`源码，可参考官方Doc：[https://ci.apache.org/projects/flink/flink-docs-release-1.7/flinkDev/building.html](https://ci.apache.org/projects/flink/flink-docs-release-1.7/flinkDev/building.html)；建议用公司内部环境（Jenkins, JDK, etc.）并结合官方Doc来完成打包。

简单列下我司打包的Jenkins配置：
`Branches to build`: refs/tags/release-1.7.1
`Repository URL`: git@gitlab.company.com:git/flink.git
`Build - Goals and options`: clean install -DskipTests -Dfast -Dhadoop.version=3.1.0
`Post Steps - Execute shell`: cd flink-dist/target/flink-1.7.1-bin
tar -czvf flink-1.7.1-hadoop31-scala_2.11.tar.gz flink-1.7.1/

**报错：**
使用`./bin/flink run -m yarn-cluster`的方式提交Job时出现如下报错：

```java

client端：
------------------------------------------------------------
 The program finished with the following exception:

java.lang.RuntimeException: Error deploying the YARN cluster
	at org.apache.flink.yarn.cli.FlinkYarnSessionCli.createCluster(FlinkYarnSessionCli.java:556)
	at org.apache.flink.yarn.cli.FlinkYarnSessionCli.createCluster(FlinkYarnSessionCli.java:72)
	at org.apache.flink.client.CliFrontend.createClient(CliFrontend.java:962)
	at org.apache.flink.client.CliFrontend.run(CliFrontend.java:243)
	at org.apache.flink.client.CliFrontend.parseParameters(CliFrontend.java:1086)
	at org.apache.flink.client.CliFrontend$2.call(CliFrontend.java:1133)
	at org.apache.flink.client.CliFrontend$2.call(CliFrontend.java:1130)
	at org.apache.flink.runtime.security.HadoopSecurityContext$1.run(HadoopSecurityContext.java:43)
	at java.security.AccessController.doPrivileged(Native Method)
	at javax.security.auth.Subject.doAs(Subject.java:422)
	at org.apache.hadoop.security.UserGroupInformation.doAs(UserGroupInformation.java:1682)
	at org.apache.flink.runtime.security.HadoopSecurityContext.runSecured(HadoopSecurityContext.java:40)
	at org.apache.flink.client.CliFrontend.main(CliFrontend.java:1130)
Caused by: java.lang.RuntimeException: Couldn`t deploy Yarn cluster
  	at org.apache.flink.yarn.AbstractYarnClusterDescriptor.deploy(AbstractYarnClusterDescriptor.java:443)
  	at org.apache.flink.yarn.cli.FlinkYarnSessionCli.createCluster(FlinkYarnSessionCli.java:554)
  	... 12 more
  Caused by: org.apache.flink.yarn.AbstractYarnClusterDescriptor$YarnDeploymentException: The YARN application unexpectedly switched to state FAILED during deployment.

  server端：
  Uncaught error from thread [flink-akka.remote.default-remote-dispatcher-5] shutting down JVM since 'akka.jvm-exit-on-fatal-error' is enabled for ActorSystem[flink]
  java.lang.VerifyError: (class: org/jboss/netty/channel/socket/nio/NioWorkerPool, method: newWorker  signature: (Ljava/util/concurrent/Executor;)Lorg/jboss/netty/channel/socket/nio/AbstractNioWorker;) Wrong return type in function
    at akka.remote.transport.netty.NettyTransport.<init>(NettyTransport.scala:295)
    at akka.remote.transport.netty.NettyTransport.<init>(NettyTransport.scala:251)
    at sun.reflect.NativeConstructorAccessorImpl.newInstance0(Native Method)

```

**原因：**
`Flink 1.3.2`依赖的`Netty3`的版本较低，与`Hadoop 3.1.0`不兼容导致的报错；不兼容的`Netty`版本为：`Netty-3.4.0.Final` 和`Netty-3.10.6.Final`；曾尝试通过修改`NioWorkerPool`的源文件并编译新的class文件以解决这个问题，但以失败告终；

**解决：**
升级`Flink`版本；经测试：`Flink 1.6, 1.7` + `Hadoop 3.1.0`编译出的包是可以正常提交给`Hadoop 3.1.0`集群的；

**结论：**
`Flink 1.3.2`与`Hadoop 3.1.0`存在兼容性问题，若需提交`Flink Job`到`Hadoop 3.1.0`及以上，最好使用`Flink 1.6+`

### Flink-Avro序列化与AllowNull
`Flink`类型系统和序列化方式可关注官方doc：
[https://ci.apache.org/projects/flink/flink-docs-master/dev/types_serialization.html](https://ci.apache.org/projects/flink/flink-docs-master/dev/types_serialization.html)
`Flink`默认支持`PoJoSerializer`，`AvroSerializer`，`KryoSerializer`三种序列化，官方推荐使用Flink自己实现的`PojoSerializer`序列化（在遇到无法用PoJoSerializer转化的时默认用`KryoSerializer`替代，具体可关注`GenericTypeInfo`），`Kryo`性能较`Avro`好些，但无供跨语言的解决方案且序列化结果的自解释性较`Avro`差；经过功能性能等多方面的调研，计划`Task`间数据传递用`Kryo`，提高数据序列化性能，状态的`checkpoint`，`savepoint`用`Avro`，让状态有更好的兼容&可迁移性；此功能从`Flink 1.7+`开始支持具体关注doc:
[https://ci.apache.org/projects/flink/flink-docs-release-1.7/dev/stream/state/custom_serialization.html](https://ci.apache.org/projects/flink/flink-docs-release-1.7/dev/stream/state/custom_serialization.html)

OK，接下来说遇到的在`Avro`下的对象序列化异常，本次bug和上边的内容无关~,~. 这个报错只是因为`Flink-Avro`不支持`null`的序列化；

**报错：**
开启`Avro`序列化（`enableForceAvro()`），`Task`间通过网络传递数据对象时，若对象内部有`field`为`null`，数据无法正常序列化抛出如下异常：

```java

java.lang.RuntimeException: in com.app.flink.model.Line in string null of string in field switchKey of com.app.flink.model.Line
	at org.apache.flink.streaming.runtime.io.RecordWriterOutput.pushToRecordWriter(RecordWriterOutput.java:104)
	at org.apache.flink.streaming.runtime.io.RecordWriterOutput.collect(RecordWriterOutput.java:83)
	at org.apache.flink.streaming.runtime.io.RecordWriterOutput.collect(RecordWriterOutput.java:41)
	... more
Caused by: java.lang.NullPointerException: in com.app.flink.model.Line in string null of string in field switchKey of com.app.flink.model.Line
	at org.apache.avro.reflect.ReflectDatumWriter.write(ReflectDatumWriter.java:145)
	at org.apache.avro.generic.GenericDatumWriter.write(GenericDatumWriter.java:58)
	at org.apache.flink.api.java.typeutils.runtime.AvroSerializer.serialize(AvroSerializer.java:135)
	... 18 more
Caused by: java.lang.NullPointerException
	at org.apache.avro.specific.SpecificDatumWriter.writeString(SpecificDatumWriter.java:65)
	at org.apache.avro.generic.GenericDatumWriter.write(GenericDatumWriter.java:76)
	at org.apache.avro.reflect.ReflectDatumWriter.write(ReflectDatumWriter.java:143)
	at org.apache.avro.generic.GenericDatumWriter.writeField(GenericDatumWriter.java:114)
	at org.apache.avro.reflect.ReflectDatumWriter.writeField(ReflectDatumWriter.java:175)
	at org.apache.avro.generic.GenericDatumWriter.writeRecord(GenericDatumWriter.java:104)
	at org.apache.avro.generic.GenericDatumWriter.write(GenericDatumWriter.java:66)
	at org.apache.avro.reflect.ReflectDatumWriter.write(ReflectDatumWriter.java:143)

```

**原因：**
`Flink`在初始化`Avro Serializer`时使用`org.apache.avro.reflect.ReflectData`构造`DatumReader`和`DatumWriter`，`ReflectData`本身不支持`null`的序列化（`Avro`有提供`ReflectData`的扩展类`org.apache.avro.reflect.AllowNull`来支持`null`的序列化，但`Flink`并未使用），从而导致`null field`的序列化异常；而`Kryo`默认支持`null field`，程序可正常运行；`Flink 1.3.2`下代码截图如下：

Avro:
![avro serializer](/images/posts/flink/avro-serializer-1.png)

Kryo:
![kryo serializer](/images/posts/flink/kryo-serializer.png)

`Flink 1.7`后对集成`Avro`的代码变更较大，但依旧会出现当前Bug，代码逻辑入口：
![kryo serializer](/images/posts/flink/avro-serializer-2.png)

**解决：**
1. 给需要序列化的`field`值赋默认值；
2. 改用`Kryo`（`enableForceKryo()`）序列化；
3. 去除`enableForceXXX()`尝试下默认的`PoJoSerializer`；

### ClassLoader导致Failover Failed

本问题在`Kafka0.8 + Flink 1.7`（`cannot guarantee End-to-End exactly once`）环境下造成了数据丢失的问题；

**报错：**
```java

异常一：kafka异常（导致了Job重启）
Caused by: java.lang.Exception: Could not complete snapshot 69 for operator wep
	at org.apache.flink.streaming.api.operators.AbstractStreamOperator.snapshotState(AbstractStreamOperator.java:422)
	at org.apache.flink.streaming.runtime.tasks.StreamTask$CheckpointingOperation.checkpointStreamOperator(StreamTask.java:1113)
	at org.apache.flink.streaming.runtime.tasks.StreamTask$CheckpointingOperation.executeCheckpointing(StreamTask.java:1055)
	at org.apache.flink.streaming.runtime.tasks.StreamTask.checkpointState(StreamTask.java:729)
	at org.apache.flink.streaming.runtime.tasks.StreamTask.performCheckpoint(StreamTask.java:641)
	at org.apache.flink.streaming.runtime.tasks.StreamTask.triggerCheckpointOnBarrier(StreamTask.java:586)
	... 8 more
Caused by: java.lang.Exception: Failed to send data to Kafka: This server is not the leader for that topic-partition.
	at org.apache.flink.streaming.connectors.kafka.FlinkKafkaProducerBase.checkErroneous(FlinkKafkaProducerBase.java:375)
	at org.apache.flink.streaming.connectors.kafka.FlinkKafkaProducerBase.snapshotState(FlinkKafkaProducerBase.java:363)
	at org.apache.flink.streaming.util.functions.StreamingFunctionUtils.trySnapshotFunctionState(StreamingFunctionUtils.java:118)
	at org.apache.flink.streaming.util.functions.StreamingFunctionUtils.snapshotFunctionState(StreamingFunctionUtils.java:99)
	at org.apache.flink.streaming.api.operators.AbstractUdfStreamOperator.snapshotState(AbstractUdfStreamOperator.java:90)
	at org.apache.flink.streaming.api.operators.AbstractStreamOperator.snapshotState(AbstractStreamOperator.java:395)
	... 13 more


异常二：Job重启，Dubbo异常
com.alibaba.dubbo.rpc.RpcException: Rpc cluster invoker for interface com.etl.commons.api.GenericService on consumer xxx.xx.xx.xx use dubbo version flink is now destroyed! Can not invoke any more.
      at com.alibaba.dubbo.rpc.cluster.support.AbstractClusterInvoker.checkWheatherDestoried(AbstractClusterInvoker.java:233)
      at com.alibaba.dubbo.rpc.cluster.support.AbstractClusterInvoker.invoke(AbstractClusterInvoker.java:215)
      at com.alibaba.dubbo.rpc.cluster.support.wrapper.MockClusterInvoker.invoke(MockClusterInvoker.java:72)
      at com.alibaba.dubbo.rpc.proxy.InvokerInvocationHandler.invoke(InvokerInvocationHandler.java:52)
      at com.alibaba.dubbo.common.bytecode.proxy0.queryBaseDimension(proxy0.java)
```

**过程：**
`Kafka`抖动或异常导致`Flink sink data to Kafka`时丢失`Topic Leader`（出现异常一），进而导致`Flink Job`重启；
`Flink Job`重启正常，`Job`状态是`running`，但`flink operator`中的`Dubbo`未初始化导致无法提供服务（在`Flink Job`重启过程中由于`DubboService.java`中代码不规范而导致`Dubbo`服务未初始化，`Dubbo`服务不可用）；
`Flink Source`开始消费数据，当需调用含有`DubboService`的`Operator`处理数据时抛异常（出现异常二），数据无法正常处理并下发到下一个`ChainOperator`；
上游数据不断被消费，但无法正常处理并下发，这就照成的`Source`端丢数的现象；

**原因：**
在`Standalone`模式下`Flink Job`的失败自动重启`Dubbo`服务可以正常初始化的，但在`OnYarn`模式下初始化`Dubbo`服务会不可用；
`Flink Standalone`下失败重启过程中，`DubboService`会先卸载再重新加载，`Dubbo`服务正常初始化；
`OnYarn`下失败重启的过程中类`DubboService`不会被JVM卸载，同时代码书写不规范，导致内存驻留了已被closed的`DubboService`；
调用已经closed的`Dubbo`服务则数据无法正常处理并抛异常；

问题代码：
```java
private static DubboService dubboService;
public static void init() {
   if (dubboService == null) {
      Dubbo.init();
      logger.info("Dubbo initialized.");
   } else {
      logger.info("Dubbo already existed.");
   }
}
// 未对dubboService做置空处理
public static void destroy() {
    dubboService.destroy();
}
```

**解决：**
在`destory`方法中清除所有`static`的共享变量，使得`job`重启时候可以正常重新初始化`Dubbo`服务，如下代码：
修改代码：
```java
public static void destroy() {
   dubboService.destroy();
   // 置空
   dubboService = null;
 }
```

**补充：**

`OnYarn`的`Flink Job`重启时候是不会释放`Container`并重新申请的，自动重启前后的`JobId`保持不变；
`Standalone`模式下，User code的是在`TaskManager`环境中加载启动的用的`ClassLoader`是：`org.apache.flink.runtime.execution.librarycache.FlinkUserCodeClassLoader`；
`OnYarn`模式下（`bin/flink run -m yarn-cluster`），`User Code`和`Flink Framework`一并提交到`Container`中，`User Code`加载用的`ClassLoader`是：`sun.misc.Launcher$AppClassLoader`；
由于`ClassLoader`不同，导致`User Code`卸载时机不同，也就导致在`OnYarn`模式下`Restart Job`，`DubboService`初始化异常；

可参考Flink Docs：https://ci.apache.org/projects/flink/flink-docs-release-1.7/monitoring/debugging_classloading.html

**注：** `coding`时候尽量用`RichFunction` 的`open()/close()`方法初始化对象，尽可能用（单例）对象持有各种资源，减少使用`static`（生命周期不可控），及时对`static`修饰对象手动置空；

### taskmanager.network.numberOfBuffers配置

解决该问题需要对Flink的Task数据交换&内存管理有深刻的理解，可参考文章：
[https://cwiki.apache.org/confluence/display/FLINK/Data+exchange+between+tasks](https://cwiki.apache.org/confluence/display/FLINK/Data+exchange+between+tasks)

**环境：**

`Job`的运行`Topology`图：
```mermaid
graph LR
A(Task-Front `parallelism:30`) -->|shuffle| D(Task-End `parallelism:90`)
```
`Task-Front`负责从Kafka读取数据、清洗、校验、格式化，其并行度为30；
`Task-End`负责数据的解析&处理，并发送到不同的`Kafka Topic`；
`Task-Front`并行度为30，`Task-End`并行度为90；
`Task-Front`通过网络IO将数据传递给`Task-End`；

Job压测环境：
`Flink 1.3.2`的`Standalone`模式；
2个`JobManager`高可用；
8个`TaskManager`（一台物理机一个`TaskManager`），主要配置如下：
```yaml
taskmanager.numberOfTaskSlots: 13
taskmanager.network.numberOfBuffers: 10240
taskmanager.memory.segment-size: 32768   # 32K
taskmanager.memory.size: 15g
```

**问题：**
在`Standalone`下由于`taskmanager.network.numberOfBuffers`配置数量过小导致`data exchange buffer`不够用，消费`Kafka`数据的速度减缓并出现大幅度的抖动，严重情况下导致`FlinkKafkaConsumer`停止消费数据，从`JMX`监控看到如下现象：
：
![buffer-1](/images/posts/flink/task-buffer-1.png)
![buffer-2](/images/posts/flink/task-buffer-2.png)
![buffer-3](/images/posts/flink/task-buffer-3.png)

**分析：**
从`JMX`监控看，主要是因为`Buffer`原因导致的线程`wait`，`cpu`抖动；
共8台机器，每台机器上有13个`Slot`，根据`Job`是`30 + 90`的并行度来看，每台机器上同时存在多个`Task-Front`和`Task-End`；
`Task-Front`需将处理好的数据存储到`ResultPartition`；
`Task-End`需将接收到的数据存储到`InputGate`以备后续处理；
`Task-End`将处理好的数据直接发送给`Kafka Topic`，所以不需要`ResultPartition`；
一个`Task-Front`需将自己处理的数据`shuffle`到90个`Task-End`；

通过`Flink Buffer`分配原理可知，需要`30*90`个`ResultPartition`，`90*30`个`InputGate`；
若平均分则每个`ResultPartition/InputGate`可分配到的`Buffer`数量是：
(10240 * 8) / (30*90 + 90 * 30) ≈ 15个(每个32k)；
在`8w+/s, 5~10k/条`的数据量下的15个`Buffer`会在很短时间被填满，造成`Buffer`紧缺，`Task-Front`无法尽快将数据`shuffle`到下游，`Task-End`无法获取足够的数据来处理；

**解决：**
增大`taskmanager.network.numberOfBuffers`数量，最好保证每个`ResultPartition/InputGate`分配到90+（压测预估值，不保证效果最佳）的`Buffer`数量；

**补充：**
在`OnYarn`下几乎不会遇到该为题，了解该问题有助于理解`Flink`内存管理&数据传递；
`Flink 1.5`之后`Buffer`默认用堆外内存，并`deprecated`了`taskmanager.network.numberOfBuffers`，用`taskmanager.network.memory.max`与`taskmanager.network.memory.min`代替；
关于`Buffer`分配&动态调整等逻辑可关注以下几个类：`TaskManagerRunner` / `Task` / `NetworkEnvironment` / `LocalBufferPool` / `InputGate` / `ResultPartition` / `MemorySegment`，在此不做源码解读，最后贴两个截图：
![buffer-4](/images/posts/flink/task-buffer-4.png)
![buffer-5](/images/posts/flink/task-buffer-5.png)

### OnYarn下log的打印

`Standalone`与`OnYarn`下的日志配置和输出路径有较大的区别，`Standalone`下直接在`maven`的`resources`目录下配置`log4j2.xml/log4j.properties`即可完成日志配置，但在`OnYarn`下 需要通过修改`${flink.dir}/conf/log4j.properties`的配置来定义日志输出，这些配置在`container.sh`启动`TaskManager/JobManager`时被加载以初始化`log4j`，配置如下：

```properties
# Log all infos in the given file
log4j.appender.file=org.apache.log4j.DailyRollingFileAppender
# ${log.file} input by luanch_container.sh
log4j.appender.file.file=${log.file}
log4j.appender.file.append=false
log4j.appender.file.encoding=UTF-8
log4j.appender.file.DatePattern='.'yyyy-MM-dd
log4j.appender.file.layout=org.apache.log4j.PatternLayout
log4j.appender.file.layout.ConversionPattern=%d{yyyy-MM-dd HH:mm:ss,SSS} %-5p %-60c %x - %m%n
```

`${flink.dir}/conf`下不同log配置文件所适用的范围如下图：
![log ui](/images/posts/flink/log-path-1.png)

`${flink.dir}`中默认不包含`DailyRollingFileAppender`的依赖，所以在使用`DailyRollingFileAppender`时还需要添加依赖`apache-log4j-extras-1.2.17.jar`到`${flink.dir}/lib/`目录，`Flink Job`的日志会输出到`Yarn Container`的日志目录下（由`yarn-site.xml`中的`yarn.nodemanager.log-dirs`指定），`Container`的启动脚本（可在`YarnUI log`中找到）示例：

```shell
// log.file 即日志输出目录
exec /bin/bash -c "$JAVA_HOME/bin/java -Xms1394m -Xmx1394m -XX:MaxDirectMemorySize=654m -XX:+UseG1GC -XX:MaxGCPauseMillis=100 -Dlog.file=/home/hadoop/apache/hadoop/latest/logs/userlogs/application_1547994202702_0027/container_e02_1547994202702_0027_01_000002/taskmanager.log -Dlogback.configurationFile=file:./logback.xml -Dlog4j.configuration=file:./log4j.properties org.apache.flink.yarn.YarnTaskExecutorRunner --configDir . 1> /home/hadoop/apache/hadoop/latest/logs/userlogs/application_1547994202702_0027/container_e02_1547994202702_0027_01_000002/taskmanager.out 2> /home/hadoop/apache/hadoop/latest/logs/userlogs/application_1547994202702_0027/container_e02_1547994202702_0027_01_000002/taskmanager.err"
```

**运行中的FlinkOnYarn Job日志查看：**
直接通过`YarnUI`界面查看每个`Container`的日志输出，如下图：
![log ui](/images/posts/flink/log-path-3.png)

在`container`所在节点的对应目录下通过`tail, less`等shell命令查看日志文件，如下图：
![log ui](/images/posts/flink/log-path-2.png)

在`hadoop`配置`enable log aggregation`时，可以通过`yarn logs -applicationId ${application_id}`获取log；

**已完成的FlinkOnYarn Job日志查看：**
在配置`hadoop log aggregation`时，可以通过`yarn logs -applicationId ${application_id}`获取log；

**注：** `Spark`中`HistoryServer`支持用户从`YarnUI`中查看已完成`Job`的日志等，但`Flink`中的`HistoryServer`在`OnYarn`下不可用，留下问题后续解决；

### Prometheus监控

`FlinkUI`的指标监控使用起来不够方便，同时在`Job`比较大时往`FlinkUI`上添加监控指标时卡顿现象非常明显，需要选择一个更好的Job监控工具来完成Job的监控任务，`Prometheus`是一个很好的选择；具体说明可参考官方Doc：[https://ci.apache.org/projects/flink/flink-docs-release-1.7/monitoring/metrics.html](https://ci.apache.org/projects/flink/flink-docs-release-1.7/monitoring/metrics.html)；
`Flink`从1.4版本开始支持`Prometheus`监控`PrometheusReporter`，在`Flink1.6`时有加入了`PrometheusPushGatewayReporter`的功能；
`PrometheusReporter`是`Prometheus`被动通过`Http`请求`Flink`集群以获取指标数据；
`PrometheusPushGatewayReporter`是`Flink`主动将指标推送到`Gateway`，`Prometheus`从`Gateway`获取指标数据；

`OnYarn`下`Flink`端开启`Prometheus`监控的配置：
```yaml
// Flink可以同时定义多个reportor，本示例定义了jmx和prometheus两种reportor
// JMX用RMI，Prom用Http
metrics.reporters: my_jmx_reporter,prom
metrics.reporter.my_jmx_reporter.class: org.apache.flink.metrics.jmx.JMXReporter
metrics.reporter.my_jmx_reporter.port: 9020-9045

metrics.reporter.prom.class: org.apache.flink.metrics.prometheus.PrometheusReporter
metrics.reporter.prom.port: 9250-9275
```

当`Job`分配到`Yarn Container`启动`JobManager/TaskManager`的过程中会实例化`metrics.reporter.xxx.class`配置的类，启动监控服务；
`PrometheusReporter`是通过暴漏Http地址来提供监控指标访问功能的，在每个`JobManager/TaskManager`启动的过程中都会实例化一个`PrometheusReporter`对象，并启动`HttpServer`服务绑定到一个端口上
（`metrics.reporter.prom.port`配置的区间从小到大第一个未占用的端口）；因此`metrics.reporter.prom.port`配置的区间可允许的最大端口数必须大于等于一台机器最大可启动的`Container`数量，
否则会有监控服务启动失败的报错，如下：

**报错：**
```java
2019-03-12 20:44:39,623 INFO  org.apache.flink.runtime.metrics.MetricRegistryImpl           - Configuring prom with {port=9250, class=org.apache.flink.metrics.prometheus.PrometheusReporter}.
2019-03-12 20:44:39,627 ERROR org.apache.flink.runtime.metrics.MetricRegistryImpl           - Could not instantiate metrics reporter prom. Metrics might not be exposed/reported.
java.lang.RuntimeException: Could not start PrometheusReporter HTTP server on any configured port. Ports: 9250
        at org.apache.flink.metrics.prometheus.PrometheusReporter.open(PrometheusReporter.java:70)
        at org.apache.flink.runtime.metrics.MetricRegistryImpl.<init>(MetricRegistryImpl.java:153)
        at org.apache.flink.runtime.taskexecutor.TaskManagerRunner.<init>(TaskManagerRunner.java:137)
        at org.apache.flink.runtime.taskexecutor.TaskManagerRunner.runTaskManager(TaskManagerRunner.java:330)
        at org.apache.flink.runtime.taskexecutor.TaskManagerRunner$1.call(TaskManagerRunner.java:301)
        at org.apache.flink.runtime.taskexecutor.TaskManagerRunner$1.call(TaskManagerRunner.java:298)
```

## 总结

本文简单总结了Flink使用过程中遇到的几个重要的问题，`taskmanager.network.numberOfBuffers配置`的问题有在stackoverflow, FlinkCommunity问过，但没得到较为准确的答案，最终通过gg文档&源码才逐渐理解，源码&Google的重要性不言而喻！

## 参考
- [https://cwiki.apache.org/confluence/display/FLINK/Data+exchange+between+tasks](https://cwiki.apache.org/confluence/display/FLINK/Data+exchange+between+tasks)
- [https://ci.apache.org/projects/flink/flink-docs-release-1.7/flinkDev/building.html](https://ci.apache.org/projects/flink/flink-docs-release-1.7/flinkDev/building.html)
- [https://ci.apache.org/projects/flink/flink-docs-release-1.7/dev/stream/state/custom_serialization.html](https://ci.apache.org/projects/flink/flink-docs-release-1.7/dev/stream/state/custom_serialization.html)
- [https://ci.apache.org/projects/flink/flink-docs-master/dev/types_serialization.html](https://ci.apache.org/projects/flink/flink-docs-master/dev/types_serialization.html)
- [https://ci.apache.org/projects/flink/flink-docs-release-1.7/monitoring/debugging_classloading.html](https://ci.apache.org/projects/flink/flink-docs-release-1.7/monitoring/debugging_classloading.html)
- [https://ci.apache.org/projects/flink/flink-docs-release-1.7/flinkDev/building.html](https://ci.apache.org/projects/flink/flink-docs-release-1.7/flinkDev/building.html)
- [https://ci.apache.org/projects/flink/flink-docs-release-1.7/monitoring/logging.html](https://ci.apache.org/projects/flink/flink-docs-release-1.7/monitoring/logging.html)
- https://ci.apache.org/projects/flink/flink-docs-release-1.7/monitoring/historyserver.html
- [https://ci.apache.org/projects/flink/flink-docs-release-1.7/monitoring/debugging_classloading.html](https://ci.apache.org/projects/flink/flink-docs-release-1.7/monitoring/debugging_classloading.html)
- [https://ci.apache.org/projects/flink/flink-docs-release-1.7/monitoring/metrics.html](https://ci.apache.org/projects/flink/flink-docs-release-1.7/monitoring/metrics.html)
- [https://ci.apache.org/projects/flink/flink-docs-release-1.7/dev/types_serialization.html](https://ci.apache.org/projects/flink/flink-docs-release-1.7/dev/types_serialization.html)
