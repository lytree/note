---
title: Checkpoint技术
date: 2023-01-23T11:26:19Z
lastmod: 2023-01-23T15:11:55Z
---

# Checkpoint技术

## 目标

> 为了解决已下问题

* 缩短数据库恢复时间；
* 缓冲池不够用时，将脏页刷新到磁盘；
* 重做日志不可用时，刷新脏页

## 实现

### Sharp CheckPoint

> 发生在数据库关闭的时候，将所有脏页都刷新回磁盘

### Fuzzy CheckPoint

> 进行部分脏页的刷新，有效循环利用Redo日志

#### Master Thread Checkpoint

　　差不多已每秒或每十秒的速度从缓存池的脏页列表中刷新一定比例的页回磁盘。此操作是异步的，不会阻塞其他操作。

#### FFLUSH_LRU_LIST Checkpoint

　　flush_lru_list checkpoint是在单独的page cleaner线程中执行的。Buffer Pool的LRU空闲列表中保留一定数量的空闲页面，来保证Buffer Pool中有足够的空间应对新的数据库请求。

　　在空闲列表不足时，发生flush_lru_list checkpoint，空闲数量阈值是可以配置的。（innbdb_lru_scan_depth）

　　5.6 后放在单独的Page Cleaner线程中进行，不会阻塞用户查询线程

#### Async/Sync Flush Checkpoint

　　在重做日志文件不可用的情况是，需要强制将一些页刷新回磁盘。

　　5.6 后放在单独的Page Cleaner线程中进行，不会阻塞用户查询线程。

　　​`​ Show Engine Innodb Status`​ 可以查看刷新次数和进行刷新的类型

#### Dirty Page too much CheckPoint

　　脏页的数量太多，导致InnoDB存储引擎强制进行Checkpoint。其目的总的来说还是为了保证缓冲池中有足够可用的页。

　　可由参数`innodb_max_dirty_pages_pct`​控制，当缓冲池中脏页的数量占据75%时，强制进行Checkpoint，刷新一部分的脏页到磁盘。
