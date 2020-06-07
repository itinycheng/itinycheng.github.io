---
layout: post
title: "产品线迁移私有云的问题汇总"
categories: flink
tags: 迁移 机房 云服务
author: tiny
---

- content
  {:toc}

## 概述

公司领导在 19 年底确定将所有服务迁移到某云服务，我所在团队负责的两条产品线需在六月前完成迁移；这两条产品线承载着公司 80%数据收集和报表计算&查询服务，指标较多且系统复杂，迁移压力略大；从 20 年初开始制定迁移方案，进行实际的迁移操作，整个迁移在团队 4 个人的全职投入和在 DBA、基础组件同事的协助下最终在 5 月中旬完成迁移。本文目的在于简单记录下自己在迁移过程中所遇到的一些技术问题。

## 说明

两条产品线主要功能有数据收集，实时数据处理，离线数据处理，指标查询四类，涉及到的需迁移的技术组件如下：

- 数据收集：`Collector（自研，Go 实现）`；
- 实时处理：`Kafka`, `Flink`, `Etl-formwork（自研 Pipeline）`；
- 离线处理：`Crontab + Hive + Python2`；
- 存储服务：`Druid`, `Hadoop`, `Cassandra`, `Mongo`, `ES`, `Mysql`, `Redis`；
- 其它服务：`Zookeeper`,`Jetty`, `Dubbo`, `Grafana`, `Prometheus`；
- 此外还有些相关组件不在本文关注范围内，例如：`Nginx`, `VIP`等；

在整个迁移过程中自己主要负责：实时/离线计算任务的迁移，实时/历史数据同步，指标核对工具的开发，保证迁移后各指标的准确无误；需迁移技术组件包括：`Kafka`, `Flink`, `Zookeeper`, `Dubbo`, `Crontab + Hive + Python`；

## 问题记录

### Dubbo 2.5.3 升级到 2.7.6

- 对`Dubbo`进行升级操作的原因：新环境中的调用`Dubbo`服务耗时较长，导致实时数据处理出现淤积的情况，在 `Github`找到一个类似的[issue](https://github.com/apache/dubbo/issues/5217)，随后对`Dubbo`进行了升级操作，问题得以解决；

- `Dubbo 2.7.6`基本可以向后兼容性`2.5.3`，只需升级`Dubbo`依赖版本，添加`Curator`依赖，然后将用户代码引入的 `Dubbo`类的路径变更为`com.apache`即可，如：`com.alibaba.dubbo.config.ApplicationConfig`变更为`org.apache.dubbo.config.ReferenceConfig`；

- 服务升级后发现在运行日志中有如下报错：`ERROR org.apache.dubbo.qos.server.Server - [DUBBO] qos-server can not bind localhost:22222`，直接在`Dubbo`配置中关闭`QOS`服务即可；

### Hive 0.13 升级 1.2.1/3.1.1

您没看错，确实是`Hive 0.13`，且`Hive SQL`提交到一个`Hadoop 0.20`的集群上~#.#～；这些批脚本对应的是一条产品线的离线指标计算的业务，这些脚本包含：`Shell`，`Python`，`Hive SQL`等脚本，并依赖了一个`Mongo`库，一个`Jetty`服务，三个`Mysql`库，八个`User Jar`，几十个`UDF`；本次迁移必须对`Hadoop/Hive`做升级，且将脚本的业务逻辑梳理清楚；这些批脚本的迁移大概用了 1 个月的时间；

#### **Upgrade to 1.2.1**

从`0.13`升级到`1.2.1`过程十分顺利，在自己定义的 `HiveCli` 初始化脚本中添加一行配置（`set hive.support.sql11.reserved.keywords=false;`）即可；主要是因为`Hive`从`1.2.0`版本开始，根据`SQL2011`标准增加了大量`reserved keyword`，通过这个配置可以保证包含`reserved keyword`的 SQL 可正常执行；参考：

- [https://issues.apache.org/jira/browse/HIVE-6617 ](https://issues.apache.org/jira/browse/HIVE-6617)
- [https://cwiki.apache.org/confluence/display/Hive/LanguageManual+DDL](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+DDL)

#### **Upgrade to 3.1.1**

最初迁移时是计划合并多个`Hadoop`集群到公司公用的`Hadoop 3.1.0`集群，对应的 Hive 版本也升级到`3.1.1`，但最终因牵动项目多（`Druid`, `Etl-Framework`），升级风险高而放弃；在`Hive`的`0.13`版本升级到`3.1.1`的测试阶段也解决了多个 SQL 迁移问题，在此做下记录。

- **hive.support.sql11.reserved.keywords 配置不可用**
  在`Hive 2.3`之后配置`set hive.support.sql11.reserved.keywords=false;`已经被移除，所以在 SQL 中有用到`reserved keyword`做标识符时需要放在反引号（``）内，以消除歧义；社区 issue: [https://issues.apache.org/jira/browse/HIVE-14872](https://issues.apache.org/jira/browse/HIVE-14872)。

- **Union 两张字段类型不一致的表**
  `Hive 3.1.1`环境下当`union all`两个字段类型不一致的列时，会收到如下报错信息：`FAILED: SemanticException 17:71 Schema of both sides of union should match: Column user_offset is of type string on first table and type int on second table. Error encountered near token 'charge_device'`；看起来是`Hive3`对`union all`两侧的数据集进行了类型一致性校验；对类型不一致的列进行`CAST(user_offset as int)`修改后 SQL 可正常执行；

- **hive.strict.checks.type.safety**
  严格类型安全检查，默认是 true，该属性不允许 `bigint 和 string 间的比较`，`bigint 和 double 间的比较`；将属性设置为 false，可以解除不允许上述两种不同类型间的比较的限制，在 SQL 的`Where`条件中经常会出现这种类型不一致的条件比较。

- **SQL 执行耗时增加：**
  `Hive 3.1.1`环境下发现个 SQL 执行特别慢，而同样 SQL 在 Hive0.13 都可以很快执行完成；经过进一步测试发现在`Hive3` 下，查询同一张表的不同字段（如下文提供的 SQL），性能差别很大，在单表数据量`40W+`的测试条件下，查询某个字段要用 2 个多小时，而查询其他字段只需`3～5分钟`；同样数据集同样的 SQL 以相同并行度在`Hive 0.13`下执行，查询时长都可保持在`3~5分钟`；在修改表数据存储格式（`STORED AS PARQUET`）后查询慢的问题可以解决，通过`hive --debug`进行`远程Debug`发现`Hive3`对数据反序列化阶段变代码变化较大，但没找到问题根源，先留下个问题和数据，以后再结合`JMX`, `Arthas`这些工具看个究竟吧，示例：

```HIVE
-- 表结构：
CREATE TABLE `tmp_logout`(
  `user` map<string,string> COMMENT 'user_info',
  `device` map<string,string> COMMENT 'device_info',
  `app` map<string,string> COMMENT 'app_info',
  `event` struct<eventType:string,attribute:map<string,string>,eventDatas:array<struct<key:string,value:string,type:string>>> COMMENT 'event_info')
PARTITIONED BY (`job_time` bigint, `timezone` int)
STORED AS TEXTFILE;

-- sql_1：执行速度快
insert overwrite directory '/user/hadoop/output/tmp_logout'
select app['product_id'] as pid from tmp_logout where job_time=20200420120000 and app['product_id'] is not NULL;

--sql_2：刚开始执行较快，但随时间增加执行越来越慢，最终执行完需要2H+，GC时间逐渐增加
insert overwrite directory '/user/hadoop/output/logout'
 select event.attribute['event_time'] as event_time from tmp_logout where job_time=20200420120000 and app['product_id'] is not NULL;
```

- **其他**
  尝试将执行引擎从`MR`切换到`Spark`上时，也发现了很多 SQL 执行报错的问题，比如：`Hive UDAF` 执行报错；在 Hive 配置`set hive.groupby.skewindata=true;`的情况下，有些`group by`的 SQL 的执行报错（经`Calcite`优化后的执行计划不一样），等等；总之，需调整的的地方还是挺多的。

## Kafka Offset 不提交导致的数据重复消费

用`Flink`写了个`Kafka To Kafka`数据拷贝工具，由于迁移前期两个机房之间的网络不太稳定，为防止数据拷贝任务频繁重启，给`Flink Job`添加了 checkpoint 失败不重启的配置：`env.getCheckpointConfig.setFailOnCheckpointingErrors(false)`，这个配置导致了数据重复消费的问题；

**现象：** `Checkpoint`失败持续了 6 个小时之久且没一次成功；查看日志发现每次 checkpoint 时候都会有如下报错信息：`java.lang.IllegalStateException: Correlation id for response (204658) does not match request (204657)`；`FlinkKafkaConsumer09`的 Offset 提交模式是`OffsetCommitMode.ON_CHECKPOINTS`；查看`Kafka-Manager`的`Consumer Offset`监控，发现一直没变化；重启 Job 发现 6 小时内的数据出现重复消费的问题。

**解决：** 在社区看到一个相同的问题：[https://issues.apache.org/jira/browse/KAFKA-4669](https://issues.apache.org/jira/browse/KAFKA-4669)；后来关闭了测试数据拷贝并将流量较大的几个`Topic`切到`kafka-mirror-maker.sh`上，问题得以暂时解决；当前 BUG 并未根除，当前问题也先遗留下来后序再看看源码吧。

## JDK7 升级到 JDK8

在升级 JDK 版本后，业务代码调用 ES 接口时报错，相关信息如下：

```java

// 调用 ES API 的用户代码
String value = String.valueOf(hits[0].field(collFields[i]).getValue());

// SearchHitField.java 被调用的方法
<V> V getValue();

// 报错信息：java.lang.ClassCastException: java.lang.Object cannot be cast to [C

// 用户代码反编译结果：
// String value = String.valueOf((char[]) hits[0].field(collFields[i]).getValue());

// 正确写法
Object obj = hits[0].field(collFields[i]).getValue();
String value = String.valueOf(obj);

```

从上述信息我们可以判断，当前这个报错是因为：在编过程中，用户代码上下文没有足够的信息给到编译器进行泛型类型的推断；贴一个文章作为学习参考：[http://lovestblog.cn/blog/2016/04/03/type-inference/?from=groupmessage](http://lovestblog.cn/blog/2016/04/03/type-inference/?from=groupmessage)；

## 其他问题

- 公司机房有高 IO 需求的服务如：`Druid Historical, Kafka, Cassandra, ES, Mysql, etc.` 都是用的`12块HDD做Raid5`；而到云服务是单独的磁盘阵列，能加多块盘，但没办法做`Raid5`，单块磁盘读写上限限制到`80M/s`，此外：听说磁盘和主机是分别存放并通过光纤链接到一起的；这使得前期压测花费了不少时间在磁盘性能测试上，在整个迁移过程中也做了几次的磁盘参数调整，或是更换高性能 SSD 的工作；

- `Mysql`, `Redis`, `Cassandra`这些服务是通过建立跨机房集群来完成实时数据迁移的，对跨机房的网络稳定性要求较高，期间由于大量历史数据跨机房拷贝导致了网络堵塞，这影响到了这些跨机房服务的正常运行；

- `Druid`, `Hadoop`, `ES`等服务是通过在云上新装集群，然后通过`distcp`, `elasticdump`命令将历史数据跨机房拷贝到云服务上的；

- 刚开始时，机房间的通信用是一根 10G 光纤，所有团队服务都走这一根光纤，互相会有影响，后又拉了根 5G 光纤；

- `Kafka`集群在负载不高的情况下，上游数据写入出现淤积的情况，运维同事帮忙调整了下网卡参数（`ethtool -L eth0 combined 2`, `ethtool -l eth0`）并重启机器后，淤积情况有所缓解；

- `Kafka 0.8.1`版本问题较多，迁移到了`Kafka 0.9`集群，`Kafka0.8.1`遇到的问题有：`recovery threads`只有一个线程且不能配置；删除`Topic`会导致`Broker`的报错日志激增，必须手动清楚`ZK`上的`Delete Topic`信息，并重启所有`Broker`；`auto.leader.rebalance.enable=false`是默认配置，会导致`Broker`压力不均衡，需修改为`true`；