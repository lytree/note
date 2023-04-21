---
title: Redis编译部署 
date: 2022-10-30T15:24:24Z
lastmod: 2022-10-30T15:52:02Z
---

# Redis编译部署 

## 编译

```bash
wget http://download.redis.io/releases/redis-5.0.14.tar.gz
wget https://download.redis.io/releases/redis-6.2.7.tar.gz
tar xzf redis-5.0.14.tar.gz
cd redis-5.0.14
make MALLOC=libc
```

### redis-server 配置

　　在`redis.conf`将`daemonize no`改为`daemonize yes`

### 远程访问 redis

　　搜索`redis.conf`把 `bind 127.0.0.1` 全部注释掉

### 修改 redis 数据库个数

　　修改`redis.conf` 中的 database
**Redis集群模式下不存在切换数据库，都是db0**

## Redis-cluster集群

> redis5 部署集群不需要redis-trib.rb

### 集群节点复制

> Redis-Cluster集群中，可以给每一个主节点添加从节点，主节点和从节点直接遵循主从模型的特性。
> 当用户需要处理更多读请求的时候，添加从节点可以扩展系统的读性能。

### 故障转移

> Redis集群的主节点内置了类似Redis Sentinel的节点故障检测和自动故障转移功能，当集群中的某个主节点下线时，集群中的其他在线主节点会注意到这一点，并对已下线的主节点进行故障转移。
> 集群进行故障转移的方法和Redis Sentinel进行故障转移的方法基本一样，不同的是，在集群里面，故障转移是由集群中其他在线的主节点负责进行的，所以集群不必另外使用Redis Sentinel。

### 集群分片策略

> Redis-cluster分片策略，是用来解决key存储位置的。
> 集群将整个数据库分为16384个槽位slot，所有key-value数据都存储在这些slot中的某一个上。一个slot槽位可以存放多个数据，key的槽位计算公式为：slot_number=crc16(key)%16384，其中crc16为16位的循环冗余校验和函数。
> 集群中的每个主节点都可以处理0个至16383个槽，当16384个槽都有某个节点在负责处理时，集群进入上线状态，并开始处理客户端发送的数据命令请求。

### 集群redirect转向

> 由于Redis集群无中心节点，请求会随机发给任意主节点；
> 主节点只会处理自己负责槽位的命令请求，其它槽位的命令请求，该主节点会返回客户端一个转向错误；
> 客户端根据错误中包含的地址和端口重新向正确的负责的主节点发起命令请求。

### 集群搭建

#### 配置文件案例

```bash
# redis.conf文件示例
bind 127.0.0.1
port 7001
daemonize yes
pidfile /var/run/redis_7001.pid
dir "./" # node.conf文件保存路径
logfile "/var/log/redis/7001/redis.log"
appendonly yes
appendfsync always
cluster-enabled yes
cluster-config-file nodes-7001.conf # 该文件中包含集群信息
```

> 注意：在哪个目录下执行启动命令，则node.conf（cluster-config）文件就会在该目录下创建

#### 创建集群

> 启动大于三个的奇数redis服务

```bash
redis-cli --cluster create ip:7001 ip:7002 127.0.0.1:7003 ip:7004 127.0.0.1:7005 127.0.0.1:7006 --cluster-replicas 1 
# --cluster-replicas 1  表示主从配置比，1表示的是1:1，前三个是主，后三个是从
```

#### 查询集群信息

#### 

> 注意：查询集群信息需要加上-c参数

```bash
redis-cli -c -h 127.0.0.1 -p 7001 cluster nodes
```

　　参数说明：

- -c：表示以集群方式连接惹redis
- -h：指定IP地址
- -p：指定端口
- cluster nodes：查询集群节点信息
- cluster info：查询集群状态信息

```bash
另一种查询方法
/redis-cli --cluster check 127.0.0.1:7001
```

### 集群管理

#### 添加新主节点

```bash
redis-cli --cluster add-node new_host:new_port existing_host:existing_port --cluster-master-id node_id 
```

#### hash槽重新分配

　　添加完新节点后，需要对新添加的主节点进行hash槽重新分配，这样该主节点才能存储数据，redis共有16384个槽。

```bash
redis-cli --cluster reshard host:port --cluster-from node_id --cluster-to node_id --cluster-slots <args> --cluster-yes 
```

#### 添加新从节点

```bash
redis-cli --cluster add-node new_host:new_port existing_host:existing_port --cluster-slave --cluster-master-id node_id 
```

#### 删除节点

```bash
redis-cli --cluster  del-node host:port node_id
```

### 注意事项

> 若是登录的是集群中的节点，需要加上-c参数；否则不用加该参数a
