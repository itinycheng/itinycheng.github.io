---
layout: post
title: "DorisDB,ClickHouse,TiDB的对比与选型分析"
categories: olap
tags: doris clickhouse tidb
author: tiny
---

- content
  {:toc}

## 系统对比

|               | DorisDB                                                                                                                                                                                                                                                                                                                       | ClickHouse                                                                                                                                                                                                                                               | TIDB                                                                                                                                                                                                                                                                                                                   |
| ------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 版本          | DorisDB-SE-1.15.2                                                                                                                                                                                                                                                                                                             | ClickHouse server version 21.6.5 revision 54448                                                                                                                                                                                                          | TiDB-v5.1.0 TiDB Server Community Edition (dev)                                                                                                                                                                                                                                                                        |
| 定位          | ROLAP<br/>MPP 执行架构<br/>支持物化视图(RollUp)<br/>使用 Index 加速查询<br/>向量化计算（AVX2 指令集）<br/>稀疏索引                                                                                                                                                                                                            | ROLAP<br/>MPP 架构的列存储库，多样化的表引擎<br/>以有序存储为核心的完全列存储实现<br/>并行扫磁盘上的数据块，降低单个查询的 latency<br/>极致向量化计算<br/>稀疏索引                                                                                       | HTAP<br/>行存储（TiKV 存储方式）<br/>列存储（TiFlash 组件实现）                                                                                                                                                                                                                                                        |
| 语言层面      | Front End: Java, Back End: C++                                                                                                                                                                                                                                                                                                | C++                                                                                                                                                                                                                                                      | TiDB: Go, TiKV: Rust, RocksDB: C++                                                                                                                                                                                                                                                                                     |
| 架构层面      | Front End:<br/>管理元数据<br/>监督管理 BE 的上下线<br/>解析 SQL，分发执行计划<br/>协调数据导入<br/><br/>Back End: <br/>接收并执行计算任务<br/>存储数据，管理副本<br/>执行 compact 任务                                                                                                                                        | Zookeeper: <br/>执行分布式 DDL；<br/>多副本之间的数据同步任务管理下发；<br/><br/>ClickHouse:<br/>数据存储查询；<br/>架构扁平化，可部署任意规模集群；                                                                                                     | TIDB 作为计算&调度层：<br/>SQL 解析/分发/谓词下推到存储层；<br/>集群信息收集，replica 管理等；<br/><br/>TiKV+RocksDB 作为数据存储层：<br/>TiKV 实现了事务支持/Raft 协议/数据读写；<br/>数据最终存放在 RocksDB；<br/>TiFlash: 列存储，用来加速查询&支持 OLAP 相关需求；<br/>TiSpark: 计算引擎，用来支持 OLAP 相关需求； |
| 系统安装&维护 | 成本低；<br/><br/>收费版本提供 DorisManager 工具来安装维护集群，免费版本需要手动安装；<br/>看官方文档说：“DorisDB 是一个自治的系统，节点的上下线，集群扩缩容都可通过一条简单的 SQL 命令来完成，数据的自动 Reblance”；<br/>看各种技术文提到 DorisDB 维护成本也是较低，但不清楚是否能达到官方文档中的效果和是否有各种异常出现； | 成本高<br/><br/>Clickhouse 需安装的组件较少，安装起来也较为简单；但集群不能自发感知机器拓扑变化，不能自动 Rebalance 数据，扩容/缩容/上下线分片、副本等都需要修改配置文件或执行 SQL 语句做数据迁移表变更的操作，有些时还需需要重启服务，维护成本高；<br/> | 成本中；<br/><br/>个人感觉 TiDB 的组件较多，<br/>计算存储相关就有 TiDB，TiKV，TiFlash，TiSpark 等；<br/>所以系统安装相对麻烦一点（需要安装的组件较多），对于扩容/缩容/升级等操作官方有提供 TiUP 组件，操作相对简单；                                                                                                   |
| 监控&报警     | 收费版本中可使用 DorisManager 提供的控报警的功能；<br/>免费版本可以通过 Prometheus+Grafana 进行监控报警；                                                                                                                                                                                                                     | Clikckhouse 支持对接 Prometheus ，可以通过 Prometheus+Grafana 进行监控报警；                                                                                                                                                                             | 官方提供一套 Prometheus+Grafana 来实现监控报警的功能；                                                                                                                                                                                                                                                                 |
| 系统升级      | 可以滚动/平滑的升级，官方有相关升级文档                                                                                                                                                                                                                                                                                       | 通过 yum 命令进行升级                                                                                                                                                                                                                                    | 通过 TiUP 实施在线/离线升级                                                                                                                                                                                                                                                                                            |
| 集群能力      | 官方文档中说集群规模可扩展到数百个节点，<br/>支持 10PB 级别的数据分析；                                                                                                                                                                                                                                                       | 官方提供案例集群规模在 300+，20 万亿+条的数据，<br/>10PB+级别的数据；                                                                                                                                                                                    | 官方文档中有说最大支持 512 个节点，<br/>集群最大容量 PB 级；                                                                                                                                                                                                                                                           |
| SQL 语句支持  | 支持标准 SQL 语法                                                                                                                                                                                                                                                                                                             | 支持大部分的标准 SQL 语法<br/>有部分非标准 SQL 语法，有一定学习成本                                                                                                                                                                                      | 支持标准 SQL 语法                                                                                                                                                                                                                                                                                                      |
| SQL 数据类型  | Number, String[1, 65533], Date, HyperLogLog, Bitmap, Array, Nested-Array                                                                                                                                                                                                                                                      | Number, String[1, ∞], Date, Bitmap, Enum, UUID, IP, Tuple, Array, Nested Array, Map, Nested Type, AggregateFunction, Lambda Expression                                                                                                                   | Number, String[1, ∞], Date, Blob, Binary, Enum, Set, Json                                                                                                                                                                                                                                                              |
| Schema 变更   | 支持对 column/rollup/index 等进行增删改的操作；                                                                                                                                                                                                                                                                               | Distributed, Merge, MergeTree 等表引擎支持 Alter 语句，可以对 column/index 等进行增删改的操作；ALTER 操作会阻塞对表的所有读写操作；                                                                                                                      | 支持对 column/index 进行增删改的操作；                                                                                                                                                                                                                                                                                 |
| 函数支持      | 支持日期/地理/字符串/聚合/窗口/bitmap/hash/cast 等函数；<br/> 支持 UDF，但需要写 C++；                                                                                                                                                                                                                                        | 支持函数较 DorisDB 丰富，复合数据类型的函数支持也很丰富，诸如：hasAny, hasAll 很是用；<br/> 支持 UDF，但需要写 C++；                                                                                                                                     | 支持的函数种类也很多，但在 JSON/复合结构的函数支持上一般；<br/> 暂不支持 UDF；                                                                                                                                                                                                                                         |
| 数据模型      | 明细模型，聚合模型，更新模型（允许定义 UNIQUE KEY，导入数据时需要将所有字段补全才能够完成更新操作）                                                                                                                                                                                                                           | 多样的表引擎提供了多样的数据聚合&分析模型 https://clickhouse.tech/docs/en/engines/table-engines/                                                                                                                                                         | TiFlash：列存储，TiSpark：                                                                                                                                                                                                                                                                                             |
| 事务支持      | 每个导入任务都是一个事务                                                                                                                                                                                                                                                                                                      | 没有完整的事务支持                                                                                                                                                                                                                                       | 支持分布式事务，提供乐观事务与悲观事务两种事务模型                                                                                                                                                                                                                                                                     |
| 写入支持      | 支持从 Mysql, Hive, HDFS, S3, Local Disk, Kafka 等系统拉取数据；<br/>支持通过 Spark, Flink, DataX, Insert into table select … 等方式写入数据；<br/>支持 CSV、ORCFile、Parquet、Json 等文件格式；                                                                                                                              | 支持从 PostgreSQL, Mysql, Hive, HDFS, Kafka, Local Disk, 等系统拉取数据；<br/>支持通过 Spark，Flink，Insert into table select …等方式写入数据；<br/>支持 CSV, TSV, Json, Protobuf, Avro, Parquet, ORC, XML…等数据格式；                                  | 支持从 Mysql, CSV, SQL 文件导入到 TiDB；<br/>支持通过 Load data…等方式写入数据；<br/>TiDB Lightning 支持 Dumpling, CSV, Amazon Aurora Parquet 等数据格式；                                                                                                                                                             |
| 更新支持      | 支持 Insert/Insert into table select...语句；<br/>不支持 update 语句，需要采用 Insert+Merge；                                                                                                                                                                                                                                 | 支持 Insert/Insert into table select...语句；<br/>支持 update 语句，通过语法 ALTER TABLE table_name UPDATE column1 = expr1 [, ...] WHERE filter_expr 实现，更新相当于重建分区，较重操作，以异步方式执行；                                                | TiDB 支持 Insert/Insert into table select...语句；<br/>TiDB 支持 Update 语句；<br/>TiFlash 可以与 TiKV 数据保持一致，通过 Raft 实现时数据同步；                                                                                                                                                                        |
| 删除支持      | 支持语句：DELETE FROM table_name [PARTITION partition_name] WHERE filter_expr                                                                                                                                                                                                                                                 | 支持语句：ALTER TABLE [db.]table [ON CLUSTER cluster] DELETE WHERE filter_expr                                                                                                                                                                           | TiDB 支持 Delete 语句                                                                                                                                                                                                                                                                                                  |
| 查询支持      | Select 语句基本符合 SQL92 标准，支持 inner join，outer join，semi join，anti join，cross join。在 inner join 条件里除了支持等值 join，还支持不等值 join， 为了性能考虑，推荐使用等值 join。其它 join 只支持等值 join。                                                                                                        | 有非标准查询语法，支持 Join 语句，详细看社区文档：<br/>https://clickhouse.tech/docs/en/sql-reference/statements/select/                                                                                                                                  | SELECT 语句与 MySQL 完全兼容；                                                                                                                                                                                                                                                                                         |
| 数据交换      | 外部表支持：Mysql, HDFS, ElasticSearch, Hive;<br/>交互支持：Kafka, Flink, Spark, Hive, Http, Mysql wire protocol                                                                                                                                                                                                              | 外部表支持：Mysql, HDFS, Http url, Jdbc, Odbc；<br/>交互支持：Flink(通过 clickhouse-jdbc 实现), Spark, Http, Jdbc, Odbc, Native Tcp, Command line, Mysql wire protocol, SQL 可视化工具                                                                   | 交互支持：通过 Flink，TiSpark 进行查询操作；<br/> 对外暴漏的是 SQL 层，可以通过 JDBC, Mysql wire protocol 进行读写;                                                                                                                                                                                                    |
| 集群模式      | Front End + Back End                                                                                                                                                                                                                                                                                                          | Zookeeper + Clickhouse                                                                                                                                                                                                                                   | TiDB + PD + TiKV + TiFlash + TiSpark                                                                                                                                                                                                                                                                                   |
| 问题遗留      | 当前不支持 update 语句，2021.7 月的新版本会支持                                                                                                                                                                                                                                                                               | optimize table $name final 性能不高，多表 join 性能一般                                                                                                                                                                                                  |                                                                                                                                                                                                                                                                                                                        |
| 简短概括      | Doris 依靠索引+Rollup+分区分桶等方案实现高效的 OLAP，支持明细查询；复合数据类型支持有限，维护成本低；                                                                                                                                                                                                                         | Clickhouse 依靠列存储+向量化+分片+多样的表引擎实现高效 OLAP，支持明细查询；功能强大，维护成本高；                                                                                                                                                        | Tidb 更多是满足了 TP 场景，对与 AP 支持不足，OLAP 能力弱于 Doris&Clickhouse，支持明细查询；官方目标是支持 100%TP 场景和 80%AP 场景，大数据量/复杂 AP 场景通过 TiSpark 解决，维护成本较低；                                                                                                                             |
| 综合对比      | DorisDB 和 Clickhouse 都是为 OLAP 而设计的系统，DorisDB 在系统运维等方面十分方便，但相对 Clickhouse 在对复合数据类型支持上不够，暂不支持 Update 操作，在数据模型支持上也稍弱于 Clickhouse，SQL 函数支持上没 ClickHouse 丰富；                                                                                                 | Clickhouse 集群拓扑变化，分片上下线等都无法自动感知并进行 Reblance，维护成本高，多表 join 性能不稳定，但从 OLAP 引擎功能来讲 Clickhouse 是三者中最强大的一个，且有一定 Update 能力（OLAP 本身不善长 Update 操作）；                                      | TiDB 的 TP 能力三者最好，AP 能力较差，毕竟大家都说是分布式 Mysql，增/删/改/事务能力强，能同步 Mysql binlog，但 AP 能力更多依靠 TiFlash/TiSpark，查询聚合耗时较其他两者长且不可控，组件较多，学习安装成本感觉较 DorisDB 高一些；                                                                                        |

## 实时人群圈选场景需求说明

1. 画像本身是一张以`user_id`为主键，标签为列名的大宽表；
2. 画像标签数据有大量的`Array`,`MAP`,`Array<Map>`等复合类型；
3. 画像标签需要实时增/删，对存储要求是：支持实时的增/删/改列的`DDL`语句；
4. 标签数据需实时增量的更新，对存储要求是：支持大数据量的实时`UPDATE`语句；
5. 人群圈选需要实时对任意标签列实施高效的组合过滤操作；

## 功能选型

### TIDB

- **优点：**

  1. 支持实时高频的`UPDATE`操作；

- **缺点：**

  1. 复合数据类型只支持`Set`/`JSON`，对`JSON`来说，其内部结构不可见，`Schema`定义不够明确；  
     `JSON`这种半结构化数据格式是以`Binary`形式进行序列化存储的（`JSON`本身不能创建索引，需要反序列化解析后才能做过滤操作？），用于圈选性能较差，为`JSON`定义的函数不够丰富，且无法通过自定义函数来补充；

  2. 数据写入当前好想只能通过 `Insert Into` 语句，性能一般，不支持直接导入`JSON`数据；`TiDB` 有`DM`工具，但只能实时同步 `Mysql Binlog`，缺少与多种其他存储介质的数据交换功能；`Spark/Flink`的`Sink Connector`好像没有，可能需要自己手动实现；

  3. 缺少和`Hadoop`生态圈的交互以及数据同步组件，可能需要通过`Jdbc`或`TiKV API`接口实现；

- **其他：**

  1. 读数据有`TiKV Client Java Api`，`Flink Source Connector`是通过这个包自己定制直接读取 `TiKV`数据的，可以做一些`Filter Push Down`；

### DorisDB

- **优点：**

  1. 集群易维护；
  2. 支持`Array`这种复合结构（当前只明细模型可用）；
  3. 多表关联查询性能良好；

- **缺点：**

  1. 不开源，`DorisDB`源码全都看不到，社区活跃度较`ClickHouse`差；

  2. 支持复合数据类型不够，遇到`Array`,`Map`或`Array<Map>`类型的标签需要打成宽表或拆表（多表`Join`性能良好）；

  3. 当前不支持`UPDATE`操作， 需要通过`INSERT OVERWRITE`模拟更新操作（官方说是 7 月版本会支持`UPDATE`）；

  4. 当前只能在`Duplicate Table`中定义`Array`类型，而圈选功能需要选择更新/聚合模型，这使得`Array`类型标签必须拆分到新表中，不太能满足圈人的功能需求；

- **其他：**

  1. 发现一个比较怪的问题，`Apache Doris`和`DorisDB`感觉上有分歧？按道理`Apache Doris`和`DorisDB`标准版应该是同一个产品定位吧，两者间的的`Flink Connector`实现代码差异较大，内置的`UDF`函数也各不相同，源码层面二者还是有一定差别的；

  2. `Apache Doris`暂时不支持符合结构数据，如：`Array`,`Map`等；

### ClickHouse

- **优点：**

  1. 支持`UPDATE`操作（但性能一般，不如`TiDB`，可以用`INSERT OVERWRITE`模拟）；

  2. 有多样的表引擎和`SQL`函数；

  3. 支持多样的复合数据类型 `Array`, `Map`, `Nested Type`（但尝试写`JSON`数据到类型为`Array<Nested>`列时失败，可能是写入方法问题？）；

  4. `OLAP`系统，查询性能较好；

- **缺点：**

  1. 集群维护成本高；
  2. 多表`Join`性能不稳定；
  3. 非标准`SQL`，有学习成本；
  4. 每个表都需要手动分区分片，在维护的表较多时，日常维护成本昂贵；

- **其他：**

  1. 当前工程用的开源协议是`Apache License Version 2.0`；
  2. 源码没贡献到`Apache`社区，项目规划走向被俄罗斯团队把控；

### 功能选型

- **结论：**
  从功能角度出发更倾向于选择`ClickHouse`；
- **原因：**
  1. `ClickHouse`和`DorisDB`在做`OLAP`的性能和功能上高于`TiDB`；
  2. `DorisDB`的主要问题是对复合数据类型的支持不够（比如`Array`），这使得很多是`Array`类型的列必须进行拆表操作，业务成本高，增加了标签数据写入和查询等业务实现的复杂度；`Update`这个必须的`Feature`功能尚未看到；函数支持没`ClickHouse`丰富，在做查询时有些过滤规则没办法实现（比如：`Array hasAny/hasAll`）；
  3. `TiDB`对复合数据类型的支持不够，只有`Set`，`JSON`这种复合结构，与`Hadoop`生态或其他外部存储结合度不高，数据的导入导出不够方便，支持导入的数据类型也不够丰富，使用上不太方便；函数支持上没`ClickHouse`丰富，没办法做`Array hasAny/hasAll`等操作；另外`TiDB`的查询性能不保证满足需求，需做测试；
  4. `ClickHouse`从功能角度来讲是最能满足用户圈选需求的系统，唯一的问题是维护成本较高，当前国内已有公司将`Clickhouse`应用到了画像场景；

## 性能测试

- **机器配置**

|            | ClickHouse                             | TiDB(No TiFlash)                             |
| ---------- | -------------------------------------- | -------------------------------------------- |
| Linux 版本 | CentOS Linux release 7.2.1511          | CentOS Linux release 7.9.2009                |
| CPU        | QEMU Virtual CPU version (cpu64-rhel6) | Intel(R) Xeon(R) Platinum 8163 CPU @ 2.50GHz |
| CPU 核数   | 8C                                     | 8C                                           |
| CPU 线程数 | 8                                      | 8                                            |
| 内存大小   | 8G                                     | 32G                                          |
| 磁盘大小   | 1T                                     | 1T                                           |

- **SQL 语句**

```SQL

-- 表`user_profile_a`数据量：4528641条
-- 表`user_profile_b`数据量：12836360条

-- SQL-1: 基本数据类型的组合过滤
select count(*) from  user_profile_b where channel_type = 'nature' and user_type = 2 and lang = 'zh_CN' and region = 'CHN' and country = 'CHN' and gender = 0;

-- SQL-2 过滤double类型数值，通过Math.random() 为每条测试数据随机生成一个score
select count(*) from user_profile_ where score > 0.2;

-- SQL-3: 数组类型的过滤
select count() from user_profile_b where hasAny(user_watch_list, ['TSLA']); -- Clickhouse
select count(*) from  user_profile_b where JSON_CONTAINS(user_watch_list, '"TSLA"', '$'); -- TIDB

-- SQL-4: 组合SQL-1, SQL-3的所有条件过滤
select count() from  user_profile_b where channel_type = 'nature' and user_type = 2 and lang = 'zh_CN' and region = 'CHN' and country = 'CHN' and gender = 0 and hasAny(user_watch_list, ['TSLA']); -- ClickHouse
select count(*) from  user_profile_b where channel_type = 'nature' and user_type = 2 and lang = 'zh_CN' and region = 'CHN' and country = 'CHN' and gender = 0 and JSON_CONTAINS(user_watch_list, '"TSLA"', '$'); -- TiDB

-- SQL-5: Join两张表，基本数据类型过滤条件
select count(*) from user_profile_b b, user_profile_a a where a.uuid = b.uuid and a.channel_type = 'nature' and a.user_type = 2 and b.lang = 'zh_CN' and b.region = 'CHN' and b.country = 'CHN' and b.gender = 0;

-- SQL-6: Join两张表，带有Array类型的过滤条件
select count() from user_profile_b b, user_profile_a a where a.uuid = b.uuid and a.channel_type = 'nature' and a.user_type = 2 and b.lang = 'zh_CN' and b.region = 'CHN' and b.country = 'CHN' and b.gender = 0 and hasAny(a.user_watch_list, ['TSLA']);  -- ClickHouse
select count(*) from user_profile_b b, user_profile_a a where a.uuid = b.uuid and a.channel_type = 'nature' and a.user_type = 2 and b.lang = 'zh_CN' and b.region = 'CHN' and b.country = 'CHN' and b.gender = 0 and JSON_CONTAINS(a.user_watch_list, '"TSLA"', '$');  -- TiDB

```

- **查询性能对比测试**

|       | ClickHouse | TiDB(No TiFlash) | Explain                                                                                                            |
| ----- | ---------- | ---------------- | ------------------------------------------------------------------------------------------------------------------ |
| SQL-1 | 0.069s     | 1.695s           | 单表简单数据类型的条件组合查询                                                                                     |
| SQL-2 | 0.029s     | 1.113s           | 单表`Double`类型数据的条件查询                                                                                     |
| SQL-3 | 1.732s     | 30.935s          | `TiDB`官方文档说："使用`Binary`格式进行序列化，对`JSON`的内部字段的查询、解析加快"，但从测试结果来看与`CK`相差较多 |
| SQL-4 | 0.229s     | 1.142s           | 应该有做`SQL`优化：过滤简单数据类型的条件前置，减少需过滤`Array`条件的数据量，查询性能较`SQL-3`提升不少            |
| SQL-5 | 0.626s     | 0.536s           | 两张表`Join`情况下`TiDB`要略好于 `CK`，看来`CK`多表`Join`性能确实不太好？                                          |
| SQL-6 | 1.243s     | 0.880s           | 两张表`Join`情况下`TiDB`要略好于 `CK`，看来`CK`多表`Join`性能确实不太好？                                          |

- 性能测试使用的都是单节点的机器，因某些原因未能在同样的软硬件基础上做性能测试；
- 此处没有将`DorisDB`纳入对比，一方面是因为`DorisDB`满足不了画像场景的业务需求，比如对`Array`等复合类型支持不够，另一方面和`DorisDB`的工作人员沟通得到了一些性能方面的测试结论，自己假设了`DorisDB`的单表性能与`ClickHouse`相当，多表`Join`性能高于`ClickHouse`；
- 本次性能测试场景并不完整，单从具体得到的结果来看，`ClickHouse`在多表`Join`方面表现一般，若有多表做复杂`Join`或大表间做`Join`操作的需求，建议做更多具体的性能测试；
- 从`Array`查询优化来讲，在`ClickHouse`中可以考虑将`Array`转`Bitmap`来优化查询；`TiDB`有`TiFlash`，但不太清楚如何处理`Array`这类复合类型，是否能满足性能需求；
- `Insert into select`语句的性能大都取决于`Select`，非特别大数据量需写入的情况下`Insert`耗时相对较少；
- `ClickHouse`的`Delete`，`Update`语句性能不高，这个需要再做测试，或用`Insert`语句替换；
- `ClickHouse`的 `optimize table $table final`性能不高，`1200W`数据测试需要`8s～14s`时间，生产上使用该操作时最好单独周期执行，比如`5min`执行一次；
