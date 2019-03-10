---
layout: post
title:  "Flink Standalone迁移OnYarn过程中遇到的若干问题记录"
categories: flink
tags:  flink yarn
author: tiny
---

* content
{:toc}


## 前言

`Flink`是由公司前辈引入，最初版本是`Flink 1.1.5`，一直以来`Flink Job`都是跑在`Standalone`集群上的，为规避`Standalone`集群中`Job`相互影响的问题（`Standalone`无法做资源隔离），前辈们又新做了一个`Flink`集群，将互相影响的`Job`拆分到不同集群中，这时团队内部有两个`Flink Standalone`集群，虽然解决了部分`Job`互相影响的问题，但新集群是和`Kafka`，`Druid`部署在同一台物理机器的（不同进程），也出现过几次不同进程互相影响的问题；这些`Flink Standalone`集群维护/升级起来也挺麻烦的，同时新`Job`越来越多，生活苦不堪言~.~；兄弟们从2018年初就喊着`OnYarn`，结果由于人力、机器、业务迭代等各种问题一拖再拖；到2018年底终于开始了这项工作，虽然只1个人力投入，经历2-3个月的时间团队的`streaming Job`已经稳定运行在`OnYarn`模式下，本文就重点记录下这段时所遇到的一些重点问题，当前生产运行版本是`Flink 1.7.1 + Hadoop 3.1.0`

## 问题汇总

### Flink与Hadoop版本兼容性问题

### Flink-avro序列化与AllowNull

### ClassLoader导致failover failed
Job auto restart

### taskmanager.numberOfTaskSlots配置

### HistoryServer在OnYarn下不可用

### Prometheus监控配置

###
## 总结

## 参考
