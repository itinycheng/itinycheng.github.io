---
layout: post
title:  "Flink Slot详解与Flink Job在Yarn下的Runtime拓扑图优化"
categories: flink
tags:  flink yarn slot
author: tiny
---

* content
{:toc}

## Intro

近期在将Flink Job从Standalone迁移OnYarn时发现Job性能较之前降低不少，迁移前有5.2W+/S的数据消费速度，迁移到Yarn后用同样的资源消费速度降为4.9W+/S，且较之前的消费速度有轻微的抖动；经过简单的问题分析和测试验证并最终解决，在保持Job分配到的资源不变的情况下减少Container数量，整加单个Container持有的CPU&Memory资源，增加每个Container持有的Slot数量来解决消费速度降低的问题；本文书写的起因是经历该问题后发现深入理解Slot&Flink Runtime Graph是十分重要且必要的；本文主要分为两个部分，第一部分详细的分析Flink Slot与Job运行关系，第二部详细的说下遇到的问题和解决方案。

## Explain Of Slot

Flink集群是由JobManager(JM), TaskManager(TM)两大组件组成的，每个JM/TM都是运行在一个独立的JVM进程中，JM相当于Master 是集群的管理节点，TM相当于Worker 是集群的工作节点，每个TM最少持有1个Slot，Slot是Flink执行Job时的最小资源分配单位，在Slot中执行着具体的Task任务。对TM而言：它持有一定数量的CPU & Memory资源，具体可通过`taskmanager.numberOfTaskSlots`, `taskmanager.heap.size`来配置，实际上`taskmanager.numberOfTaskSlots`只是指定TM的Slot数量并不能隔离固定数量的CPU给TM使用，在不考虑slotSharing(下文详诉)的情况下一个Slot内运行着一个SubTask(Task实现Runable，SubTask可看做一个执行Task的具体实例)，所以官方建议`taskmanager.numberOfTaskSlots`配置的Slot数量和CPU相等或成比例；当然，我们可以借助Yarn等调度系统用Flink On Yarn的模式来为Yarn Container分配特定数量的CPU资源以达到严格的CPU隔离；`taskmanager.heap.size`用来配置TM的Memory，如果一个TM有N个Slot，则每个Slot分配到的Memory数量为整个TM Memory数量的1/N，同一个TM内的Slot间只有Memory隔离，CPU是共享的；对Job而言：一个Job所需的Slot数量大于等于Operator配置的最大Parallelism数，在Operator-Chain(下文详诉)的前提下Job所需的Slot数量与Job中Operator配置的最大Parallelism相等。

关于TM/Slot之间的关系可以参考如下从官方文档截取到的三张图：

**图一：** Flink On Yarn的Job提交过程，从图中我们可以了解到每个JM/TM实例都分属于不同的Yarn Container，且每个Yarn Container内只会有一个JM或TM实例；通过对Yarn的学习我们可以了解到每个Yarn Container都是一个独立的进程，一台物理机可以有多个Yarn Container存在（多个进程），每个Yarn Container都持有一定数量的CPU & Memory资源，而且是资源隔离的，进程间不共享，这就可以保证同一台机器上的多个TM之间是资源隔离的（Standalone模式下同一台机器下若有多个TM是做不到TM之间的CPU资源隔离）。
![flink on yarn](/images/posts/flink/flink_on_yarn.svg)

**图二：** Flink Job运行图，主要表达了Job提交&运行以及JM/TM进程间的心跳通信等，这里我们主要关注TM内部的构成，当前图中有两个TM，各自有3个Slot，2个Slot有Task在执行，1个Slot空闲；若这两个TM在不同Yarn Container或物理机上则拥有的CPU & Memory是相互隔离的，而TM内多个Slot之间是各自拥有1/3 TM的Memory，共享TM的CPU & 网络（Tcp：Akka, Netty服务等），心跳信息，Flink结构化的数据集等，这有效的降低了每个任务的开销。
![processes](/images/posts/flink/processes.svg)

**图三：** Task Slot的内部结构图，Slot内运行着具体的Task，它是在线程中执行的Runable对象（每个虚线框代表一个线程），这些Task实例在源码中对应的类是`Task.java`；每个Task都是由一组operators chain在一起的工作集合，将在Flink Job看成一张DAG图的话，Task代表DAG上的一个顶点（vertex），顶点之间通过数据传递方式相互链接构成整个Job运行的DAG Graph。
![processes](/images/posts/flink/tasks_slots.svg)

## Operator Chain
Operator Chain是指将Job中的Operators按照一定策略（single output operator可以chain在一起）链接起来并放置在一个Task线程中执行，Operator Chain默认开启，可通过`StreamExecutionEnvironment.disableOperatorChaining()`关闭，Flink Operator类似Storm中的Bolt，在Strom中上游Bolt到下游需要经过网络上的数据传递，而Flink的Operator Chain将多个Operator链接到一起执行，减少了数据传递/线程切换等环节，减少了系统开销的同时增加了资源利用率和Job性能，实际开发过程中需要开发者了解这些原理并能合理分配Memory & CPU给到每个Task线程；

具体Operator Chain效果可参考如下从官方文档截图：

**图四：** 图的上半部分是一个Job较为简练的视角（只有Task类别，无并行度），如图：Job Runtime期有三种类型的Task，分别是`Source->Map`, `keyBy/window/apply`, `Sink`，其中`Source->Map`是`Source()`和`Map()`chaining在一起的Task；图的下半部分是一个Job Runtime期的实际状态，Job最大的并行度为2，有5个SubTask（即5个执行线程）；若没有Operator Chain，则`Source()`和`Map()`分属不同的Thread，Task数/线程数会增加到7，线程切换/Buffer使用等较之前有所增加，处理延迟&性能较之前差。补充：在Slot Sharing Group用相同或默认组名时，当前Job所需2个slot（与Job最大Parallelism相等）。
![/images/posts/flink/tasks_chains.png](/images/posts/flink/tasks_chains.svg)


## Slot Sharing
Slot Sharing是指同一个Job下所有Task的SubTask可以运行在同一个Slot下，在这种情况下，一个Slot有可能持有Job的一整条pipeline，这也是上文我们提到的在默认slotSharing的条件下Job启动所需的Slot数和Job中Operator的最大parallelism相等的原因；通过Slot Sharing机制可以更进一步提高了资源利用率，在Slot数不变的情况下为Job设置可允许的最大的并行度，让类似window这种消耗资源的Task可以公平的分布在不同TaskManager上，同时像map/filter这种较简单的操作也不会独占Slot资源，降低资源浪费的可能性。

具体Slot Sharing效果可参考如下官方文档截图：

**图五：**
图的左下角是一个`soure-map-reduce`模型的Job，source和map是`4 parallelism`，reduce是`3 parallelism`，总计11个SubTask；这个Job最大parallelism是4，所以将这个Job发布到左侧上面的两个TM上时得到图右侧的运行图，一共占用四个Slot，有三个Slot拥有完整的`source-map-reduce`模型的pipeline，如右侧图所示；注：`map`的结果会`shuffle`到`reduce`端，右侧图的箭头只是说Slot内数据Pipline，没画出Job的数据`shuffle`过程。
![slots](/images/posts/flink/slots.svg)


![/images/posts/flink/tasks_chains.png](/images/posts/flink/slot_sharing.svg)


## 资源分配
这种资源分配也存在一定弱点，必须要保证每个task对资源的消费是一样的；

## API



![slots parallelism](/images/posts/flink/slots_parallelism.svg)


Task与SubTask的理解：


chain only exist in stream

explain chain

slotsharing
slotgroup
disablechain
startnewchain
disableoperatorchain

singleStreamOperator

slot与paramell关系

slot与task之间的关系

slot sharing

task与subTask关系

同一个Job内部多个task slotsharing

一个job所需并行度取决于最大task所需并行度；

一个slot内部可能存在多个task，因为有slotsharing的存在；

chainStatgy只在stream中存在

## Job Runtime Graph Tuning
