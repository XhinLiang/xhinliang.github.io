title: 常见数据库简介
date: 2018-11-18
tags: [Computer]
categories: Computer
toc: true
---

# Overview

![overview](/uploads/persister-frequent-db-intro-overview-548706-637199619492944531-16x9.jpg)

数据库大概是后端程序员最常用的中间件之一，今天我们来聊聊常见的数据库。

## MySQL 派系

MySQL 无疑是世界上最热门的数据库之一。也正因为足够热门，它衍生出了不少分支，各自都有一些不同的特点。

### MySQL

在国内，MySQL 是最常见的数据库之一，也是 MySQL 派系中最主流的分支，目前由 Oracle 公司维护。

特点：
1. 源代码使用 C 和 C++ 编写，性能稳定。
2. 支持多种数据库引擎（MyISAM、InnoDB、Memory、MyRocks 等），可以满足不同场景下的需要。
3. InnoDB 支持事务，但 MyISAM 不支持。
4. 索引使用 B+ 树实现，很多操作以顺序读取为主，对 HDD 更友好。
5. MyISAM 中每个索引都是一级索引；而 InnoDB 中除主键外的索引都是二级索引（先通过二级索引找到主键，再由主键定位数据）。
6. 分为社区版（免费）和商业版（收费）两种授权模式。
7. 支持主从配置：主库读写，从库只读。

### MariaDB

MariaDB 是 MySQL 的一个开源分支，目标是实现对 MySQL 100% 的兼容。
特点：
1. 它的存储引擎与 MySQL 并不完全一致。
2. 它独特的存储引擎叫 Maria，是 InnoDB 的变体，支持事务；此外还支持 FederatedX、XtraDB 等其他存储引擎。
3. 与 MySQL 100% 兼容：
   1. 数据和表定义文件（.frm）是二进制兼容的。
   2. 所有客户端 API、协议和结构都是完全一致的。
   3. 所有文件名、二进制、路径、端口等都是一致的。
4. 整体性能与 MySQL 类似。

### TiDB

TiDB 是 PingCAP 公司推出的开源分布式关系型数据库。
简介可参考官网：https://pingcap.com/docs-cn/

特点：
- 基本兼容 MySQL 协议（但不是 100% 兼容）。
- 支持分布式事务（这一点很强）。
- 支持在线 DDL。
- 100% 支持标准 ACID 事务。
- 不同于 MySQL 的主从复制方案，基于 Raft 的多数派选举协议可以提供金融级的 100% 数据强一致性保证；且在不丢失大多数副本的前提下，还能实现故障自动恢复（auto-failover），无需人工介入（这点也很强）。
- 也支持大多数 OLAP 场景（OLAP：联机分析处理）。

### AliSQL

AliSQL 是另一个 MySQL 的分支版本，目前由 Alibaba 维护。[GitHub](https://github.com/alibaba/AliSQL)

官方描述是：「在通用基准测试场景下，AliSQL 版本比 MySQL 官方版本有着 70% 的性能提升；在秒杀场景下，性能提升 100 倍。」

特点：
- 100% 兼容 MySQL。
- 也是另一套存储引擎。
- 虽然由 Alibaba 维护，但国内相关资料极少。
- 开源声势很大，但源代码已经很久没有更新（估计内部仍在开发，对外开放的版本相对滞后）。

## PostgreSQL

PostgreSQL 和 MySQL 类似，也是关系型数据库；但它还有一个特点：它是对象关系数据库管理系统（ORDBMS）。

提到 PostgreSQL，就不得不把它和 MySQL 对比一下，[这里有简单的对比](https://blog.csdn.net/tiandao2009/article/details/79839037)。

特点：
- 更偏学院派（这是个哲学问题，这里不展开）。
- 多进程架构；相比之下，MySQL 是多线程的。
- 支持同步、异步、半同步的 replica，属于物理复制。
- JOIN 性能相比 MySQL 有明显优势。

## MongoDB

MongoDB 不是关系型数据库，而是一种 NoSQL。
特点：
- 天生支持分布式，对扩展友好。
- 不支持 ACID 事务，但作为替代，有个新的概念：BASE。基本可用（Basically Available）、软状态/柔性事务（Soft state）、最终一致性（Eventual consistency）。
- 在一些场景下，基本可以替代 MySQL 使用。

## HBase

HBase 是建立在 HDFS 之上的分布式面向列数据库，也是 Google 著名论文《BigTable》的开源实现。

[这篇文章讲得不错](https://blog.csdn.net/nosqlnotes/article/details/79647096)。

HBase 是面向列的数据库，在表中按行排序。表模式定义只能列族，也就是键值对。一个表可以有多个列族，每个列族又可以有任意数量的列。后续列的值会连续地存储在磁盘上。表中的每个单元格值都具有时间戳。

HBase 常被用来存放结构简单但数据量非常大的数据。

特点：
- 较大的表中能快速查找。
- 数十亿条记录低延迟访问单个行记录（随机存取）。
- 面向列的数据库。
- 不具有固定列模式的概念，仅定义列族，每个列族可以包含多个列。
- 大宽表，列值是稀疏的，而且是半结构化的数据。
- 不支持任何事务。
- 索引能力有限。
- 支持自动故障恢复。

## InfluxDB

InfluxDB 是一个开源的时序数据库，使用 Golang 开发，特别适合处理和分析资源监控数据这类时序相关数据。

InfluxDB 自带多种特殊函数，如求标准差、随机取样数据、统计数据变化比等，使数据统计和实时分析变得更方便。

特点：
- 数据可以被标记，从而支持非常灵活的查询。
- 支持一部分 SQL 语句。
- 适合 OLAP 或监控需求。

## OpenTSDB

也是一种时序数据库，但和 InfluxDB 不同的是，它依赖 HBase 实现。

## GpDB -- Greenplum

Greenplum 数据库（也叫 GPDB）是一个分布式数据库，也是数据仓库的快速查询工具。

特点：
- 支持 SQL。
- 支持分布式事务。
- 支持线性扩展。

## ClickHouse

近年来兴起的一个 OLAP 数据库，由俄罗斯公司 Yandex 开发，性能强劲。

## LevelDb

LevelDb 是 Google 实现的高效 KV 数据库，能够支持 billion 级别的数据量。在这个数量级别下仍能保持很高的性能，主要归功于它良好的设计，尤其是 LSM 算法。

## RocksDb

RocksDB 是 Facebook 基于 C++ 编写的嵌入式 KV 存储引擎，其键和值都允许使用二进制流，并提供向后兼容的 LevelDB API。

RocksDB 针对 Flash 存储进行优化，延迟极小。RocksDB 使用 LSM 存储引擎，纯 C++ 编写。Java 版本 RocksJava 正在开发中。

我理解 RocksDb 应该是 LevelDb 的另一种实现，从功能上算是超集吧。

## MyRocks

MySQL 兼容的 RocksDb，底层实现基本与 RocksDB 一致，但作为一种存储引擎在 MySQL 中使用。国内有一些公司在使用。
