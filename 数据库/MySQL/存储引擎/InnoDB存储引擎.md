---
title: InnoDB存储引擎
date: 2023-01-21T16:25:44Z
lastmod: 2023-01-25T16:17:00Z
---

# InnoDB存储引擎

## 体系

### 后台进程

#### Master Thread

> 负责处理缓冲池中的数据异步刷新

#### IO Thread

> 负责处理IO请求的回调

#### Purge Thread

> 1.2版本以后支持设置多个

#### Page Cleaner Thread

> 1.2.x版本引入

## 插入缓存

## 自适应Hash索引

## 启动，关闭和恢复
