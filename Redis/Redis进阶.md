---
title: Redis进阶
date: 2022-10-30T14:57:14Z
lastmod: 2022-10-30T15:29:14Z
---

# Redis进阶

---

## 事务

### DISCARD

　　**DISCARD**

　　取消事务，放弃执行事务块内的所有命令。

　　如果正在使用 `WATCH` 命令监视某个(或某些) key，那么取消所有监视，等同于执行命令 `UNWATCH` 。

　　**可用版本：** >= 2.0.0

　　**时间复杂度：**

　　O(1)。

　　**返回值：**

　　总是返回 `OK` 。

```
redis> MULTI
OK

redis> PING
QUEUED

redis> SET greeting "hello"
QUEUED

redis> DISCARD
OK
```

### EXEC

　　**EXEC**

　　执行所有事务块内的命令。

　　假如某个(或某些) key 正处于 `WATCH` 命令的监视之下，且事务块中有和这个(或这些) key 相关的命令，那么 `EXEC` 命令只在这个(或这些) key 没有被其他命令所改动的情况下执行并生效，否则该事务被打断(abort)。

　　**可用版本：** >= 1.2.0

　　**时间复杂度：**

　　事务块内所有命令的时间复杂度的总和。

　　**返回值：**

　　| 事务块内所有命令的返回值，按命令执行的先后顺序排列。

　　| 当操作被打断时，返回空值 `nil` 。

```
# 事务被成功执行

redis> MULTI
OK

redis> INCR user_id
QUEUED

redis> INCR user_id
QUEUED

redis> INCR user_id
QUEUED

redis> PING
QUEUED

redis> EXEC
1) (integer) 1
2) (integer) 2
3) (integer) 3
4) PONG


# 监视 key ，且事务成功执行

redis> WATCH lock lock_times
OK

redis> MULTI
OK

redis> SET lock "huangz"
QUEUED

redis> INCR lock_times
QUEUED

redis> EXEC
1) OK
2) (integer) 1


# 监视 key ，且事务被打断

redis> WATCH lock lock_times
OK

redis> MULTI
OK

redis> SET lock "joe"        # 就在这时，另一个客户端修改了 lock_times 的值
QUEUED

redis> INCR lock_times
QUEUED

redis> EXEC                  # 因为 lock_times 被修改， joe 的事务执行失败
(nil)
```

### MULTI

　　**MULTI**

　　标记一个事务块的开始。

　　事务块内的多条命令会按照先后顺序被放进一个队列当中，最后由 `EXEC` 命令原子性(atomic)地执行。

　　**可用版本：** >= 1.2.0

　　**时间复杂度：**

　　O(1)。

　　**返回值：**

　　总是返回 `OK` 。

```
redis> MULTI            # 标记事务开始
OK

redis> INCR user_id     # 多条命令按顺序入队
QUEUED

redis> INCR user_id
QUEUED

redis> INCR user_id
QUEUED

redis> PING
QUEUED

redis> EXEC             # 执行
1) (integer) 1
2) (integer) 2
3) (integer) 3
4) PONG
```

### UNWATCH

　　**UNWATCH**

　　取消 `WATCH` 命令对所有 key 的监视。

　　如果在执行 `WATCH` 命令之后， `EXEC` 命令或 `DISCARD` 命令先被执行了的话，那么就不需要再执行 `UNWATCH` 了。

　　因为 `EXEC` 命令会执行事务，因此 `WATCH` 命令的效果已经产生了；而 `DISCARD` 命令在取消事务的同时也会取消所有对 key 的监视，因此这两个命令执行之后，就没有必要执行 `UNWATCH` 了。

　　**可用版本：** >= 2.2.0

　　**时间复杂度：**

　　O(1)

　　**返回值：**

　　总是 `OK` 。

```
redis> WATCH lock lock_times
OK

redis> UNWATCH
OK
```

### WATCH

　　**WATCH key [key ...]**

　　监视一个(或多个) key ，如果在事务执行之前这个(或这些) key 被其他命令所改动，那么事务将被打断。

　　**可用版本：** >= 2.2.0

　　**时间复杂度：**

　　O(1)。

　　**返回值：**

　　总是返回 `OK` 。

```
redis> WATCH lock lock_times
OK
```

## 客户端与服务器

### AUTH

　　**AUTH password**

　　通过设置配置文件中 `requirepass` 项的值(使用命令 `CONFIG SET requirepass password` )，可以使用密码来保护 Redis 服务器。

　　如果开启了密码保护的话，在每次连接 Redis 服务器之后，就要使用 `AUTH` 命令解锁，解锁之后才能使用其他 Redis 命令。

　　如果 `AUTH` 命令给定的密码 `password` 和配置文件中的密码相符的话，服务器会返回 `OK` 并开始接受命令输入。

　　另一方面，假如密码不匹配的话，服务器将返回一个错误，并要求客户端需重新输入密码。

> warning 因为 Redis 高性能的特点，在很短时间内尝试猜测非常多个密码是有可能的，因此请确保使用的密码足够复杂和足够长，以免遭受密码猜测攻击。

　　**可用版本：** >= 1.0.0

　　**时间复杂度：**

　　O(1)

　　**返回值：**

　　密码匹配时返回 `OK` ，否则返回一个错误。

```
# 设置密码

redis> CONFIG SET requirepass secret_password   # 将密码设置为 secret_password
OK

redis> QUIT                                     # 退出再连接，让新密码对客户端生效

[huangz@mypad]$ redis

redis> PING                                     # 未验证密码，操作被拒绝
(error) ERR operation not permitted

redis> AUTH wrong_password_testing              # 尝试输入错误的密码
(error) ERR invalid password

redis> AUTH secret_password                     # 输入正确的密码
OK

redis> PING                                     # 密码验证成功，可以正常操作命令了
PONG


# 清空密码

redis> CONFIG SET requirepass ""   # 通过将密码设为空字符来清空密码
OK

redis> QUIT

$ redis                            # 重新进入客户端

redis> PING                        # 执行命令不再需要密码，清空密码操作成功
PONG
```

### QUIT

　　**QUIT**

　　请求服务器关闭与当前客户端的连接。

　　一旦所有等待中的回复(如果有的话)顺利写入到客户端，连接就会被关闭。

　　**可用版本：** >= 1.0.0

　　**时间复杂度：**

　　O(1)

　　**返回值：**

　　总是返回 `OK` (但是不会被打印显示，因为当时 Redis-cli 已经退出)。

```
$ redis

redis> QUIT

$
```

### INFO

　　**INFO [section]**

　　以一种易于解释（parse）且易于阅读的格式，返回关于 Redis 服务器的各种信息和统计数值。

　　通过给定可选的参数 `section` ，可以让命令只返回某一部分的信息：

- `server` 部分记录了 Redis 服务器的信息，它包含以下域：
  - `redis_version` : Redis 服务器版本
  - `redis_git_sha1` : Git SHA1
  - `redis_git_dirty` : Git dirty flag
  - `os` : Redis 服务器的宿主操作系统
  - `arch_bits` : 架构（32 或 64 位）
  - `multiplexing_api` : Redis 所使用的事件处理机制
  - `gcc_version` : 编译 Redis 时所使用的 GCC 版本
  - `process_id` : 服务器进程的 PID
  - `run_id` : Redis 服务器的随机标识符（用于 Sentinel 和集群）
  - `tcp_port` : TCP/IP 监听端口
  - `uptime_in_seconds` : 自 Redis 服务器启动以来，经过的秒数
  - `uptime_in_days` : 自 Redis 服务器启动以来，经过的天数
  - `lru_clock` : 以分钟为单位进行自增的时钟，用于 LRU 管理
- `clients` 部分记录了已连接客户端的信息，它包含以下域：
  - `connected_clients` : 已连接客户端的数量（不包括通过从属服务器连接的客户端）
  - `client_longest_output_list` : 当前连接的客户端当中，最长的输出列表
  - `client_longest_input_buf` : 当前连接的客户端当中，最大输入缓存
  - `blocked_clients` : 正在等待阻塞命令（BLPOP、BRPOP、BRPOPLPUSH）的客户端的数量
- `memory` 部分记录了服务器的内存信息，它包含以下域：
  - `used_memory` : 由 Redis 分配器分配的内存总量，以字节（byte）为单位
  - `used_memory_human` : 以人类可读的格式返回 Redis 分配的内存总量
  - `used_memory_rss` : 从操作系统的角度，返回 Redis 已分配的内存总量（俗称常驻集大小）。这个值和 `top` 、 `ps` 等命令的输出一致。
  - `used_memory_peak` : Redis 的内存消耗峰值（以字节为单位）
  - `used_memory_peak_human` : 以人类可读的格式返回 Redis 的内存消耗峰值
  - `used_memory_lua` : Lua 引擎所使用的内存大小（以字节为单位）
  - `mem_fragmentation_ratio` : `used_memory_rss` 和 `used_memory` 之间的比率
  - `mem_allocator` : 在编译时指定的， Redis 所使用的内存分配器。可以是 libc 、 jemalloc 或者 tcmalloc 。

　　| 在理想情况下， `used_memory_rss` 的值应该只比 `used_memory` 稍微高一点儿。

　　| 当 `rss > used` ，且两者的值相差较大时，表示存在（内部或外部的）内存碎片。

　　| 内存碎片的比率可以通过 `mem_fragmentation_ratio` 的值看出。

　　| 当 `used > rss` 时，表示 Redis 的部分内存被操作系统换出到交换空间了，在这种情况下，操作可能会产生明显的延迟。

　　Because Redis does not have control over how its allocations are mapped to memory pages, high `used_memory_rss` is often the result of a spike in memory usage.

　　| 当 Redis 释放内存时，分配器可能会，也可能不会，将内存返还给操作系统。

　　| 如果 Redis 释放了内存，却没有将内存返还给操作系统，那么 `used_memory` 的值可能和操作系统显示的 Redis 内存占用并不一致。

　　| 查看 `used_memory_peak` 的值可以验证这种情况是否发生。

- `persistence` 部分记录了跟 `RDB` 持久化和 `AOF` 持久化有关的信息，它包含以下域：
  - `loading` : 一个标志值，记录了服务器是否正在载入持久化文件。
  - `rdb_changes_since_last_save` : 距离最近一次成功创建持久化文件之后，经过了多少秒。
  - `rdb_bgsave_in_progress` : 一个标志值，记录了服务器是否正在创建 RDB 文件。
  - `rdb_last_save_time` : 最近一次成功创建 RDB 文件的 UNIX 时间戳。
  - `rdb_last_bgsave_status` : 一个标志值，记录了最近一次创建 RDB 文件的结果是成功还是失败。
  - `rdb_last_bgsave_time_sec` : 记录了最近一次创建 RDB 文件耗费的秒数。
  - `rdb_current_bgsave_time_sec` : 如果服务器正在创建 RDB 文件，那么这个域记录的就是当前的创建操作已经耗费的秒数。
  - `aof_enabled` : 一个标志值，记录了 AOF 是否处于打开状态。
  - `aof_rewrite_in_progress` : 一个标志值，记录了服务器是否正在创建 AOF 文件。
  - `aof_rewrite_scheduled` : 一个标志值，记录了在 RDB 文件创建完毕之后，是否需要执行预约的 AOF 重写操作。
  - `aof_last_rewrite_time_sec` : 最近一次创建 AOF 文件耗费的时长。
  - `aof_current_rewrite_time_sec` : 如果服务器正在创建 AOF 文件，那么这个域记录的就是当前的创建操作已经耗费的秒数。
  - `aof_last_bgrewrite_status` : 一个标志值，记录了最近一次创建 AOF 文件的结果是成功还是失败。

　　如果 AOF 持久化功能处于开启状态，那么这个部分还会加上以下域：

- `aof_current_size` : AOF 文件目前的大小。
- `aof_base_size` : 服务器启动时或者 AOF 重写最近一次执行之后，AOF 文件的大小。
- `aof_pending_rewrite` : 一个标志值，记录了是否有 AOF 重写操作在等待 RDB 文件创建完毕之后执行。
- `aof_buffer_length` : AOF 缓冲区的大小。
- `aof_rewrite_buffer_length` : AOF 重写缓冲区的大小。
- `aof_pending_bio_fsync` : 后台 I/O 队列里面，等待执行的 `fsync` 调用数量。
- `aof_delayed_fsync` : 被延迟的 `fsync` 调用数量。
- `stats` 部分记录了一般统计信息，它包含以下域：
  - `total_connections_received` : 服务器已接受的连接请求数量。
  - `total_commands_processed` : 服务器已执行的命令数量。
  - `instantaneous_ops_per_sec` : 服务器每秒钟执行的命令数量。
  - `rejected_connections` : 因为最大客户端数量限制而被拒绝的连接请求数量。
  - `expired_keys` : 因为过期而被自动删除的数据库键数量。
  - `evicted_keys` : 因为最大内存容量限制而被驱逐（evict）的键数量。
  - `keyspace_hits` : 查找数据库键成功的次数。
  - `keyspace_misses` : 查找数据库键失败的次数。
  - `pubsub_channels` : 目前被订阅的频道数量。
  - `pubsub_patterns` : 目前被订阅的模式数量。
  - `latest_fork_usec` : 最近一次 `fork()` 操作耗费的毫秒数。
- `replication` : 主/从复制信息
  - `role` : 如果当前服务器没有在复制任何其他服务器，那么这个域的值就是 `master` ；否则的话，这个域的值就是 `slave` 。注意，在创建复制链的时候，一个从服务器也可能是另一个服务器的主服务器。

　　如果当前服务器是一个从服务器的话，那么这个部分还会加上以下域：

- `master_host` : 主服务器的 IP 地址。
- `master_port` : 主服务器的 TCP 监听端口号。
- `master_link_status` : 复制连接当前的状态， `up` 表示连接正常， `down` 表示连接断开。
- `master_last_io_seconds_ago` : 距离最近一次与主服务器进行通信已经过去了多少秒钟。
- `master_sync_in_progress` : 一个标志值，记录了主服务器是否正在与这个从服务器进行同步。

　　如果同步操作正在进行，那么这个部分还会加上以下域：

- `master_sync_left_bytes` : 距离同步完成还缺少多少字节数据。
- `master_sync_last_io_seconds_ago` : 距离最近一次因为 SYNC 操作而进行 I/O 已经过去了多少秒。

　　如果主从服务器之间的连接处于断线状态，那么这个部分还会加上以下域：

- `master_link_down_since_seconds` : 主从服务器连接断开了多少秒。

　　以下是一些总会出现的域：

- `connected_slaves` : 已连接的从服务器数量。

　　对于每个从服务器，都会添加以下一行信息：

- `slaveXXX` : ID、IP 地址、端口号、连接状态
- `cpu` 部分记录了 CPU 的计算量统计信息，它包含以下域：
  - `used_cpu_sys` : Redis 服务器耗费的系统 CPU 。
  - `used_cpu_user` : Redis 服务器耗费的用户 CPU 。
  - `used_cpu_sys_children` : 后台进程耗费的系统 CPU 。
  - `used_cpu_user_children` : 后台进程耗费的用户 CPU 。
- `commandstats` 部分记录了各种不同类型的命令的执行统计信息，比如命令执行的次数、命令耗费的 CPU 时间、执行每个命令耗费的平均 CPU 时间等等。对于每种类型的命令，这个部分都会添加一行以下格式的信息：
  - `cmdstat_XXX:calls=XXX,usec=XXX,usecpercall=XXX`
- `cluster` 部分记录了和集群有关的信息，它包含以下域：
  - `cluster_enabled` : 一个标志值，记录集群功能是否已经开启。
- `keyspace` 部分记录了数据库相关的统计信息，比如数据库的键数量、数据库已经被删除的过期键数量等。对于每个数据库，这个部分都会添加一行以下格式的信息：
  - `dbXXX:keys=XXX,expires=XXX`

　　除上面给出的这些值以外， `section` 参数的值还可以是下面这两个：

- `all` : 返回所有信息
- `default` : 返回默认选择的信息

　　当不带参数直接调用 `INFO` 命令时，使用 `default` 作为默认参数。

```
不同版本的 Redis 可能对返回的一些域进行了增加或删减。

因此，一个健壮的客户端程序在对 `info` 命令的输出进行分析时，应该能够跳过不认识的域，并且妥善地处理丢失不见的域。
```

　　**可用版本：** >= 1.0.0

　　**时间复杂度：**

　　O(1)

　　**返回值：**

　　具体请参见下面的测试代码。

```
redis> INFO
# Server
redis_version:2.9.11
redis_git_sha1:937384d0
redis_git_dirty:0
redis_build_id:8e9509442863f22
redis_mode:standalone
os:Linux 3.13.0-35-generic x86_64
arch_bits:64
multiplexing_api:epoll
gcc_version:4.8.2
process_id:4716
run_id:26186aac3f2380aaee9eef21cc50aecd542d97dc
tcp_port:6379
uptime_in_seconds:362
uptime_in_days:0
hz:10
lru_clock:1725349
config_file:

# Clients
connected_clients:1
client_longest_output_list:0
client_biggest_input_buf:0
blocked_clients:0

# Memory
used_memory:508536
used_memory_human:496.62K
used_memory_rss:7974912
used_memory_peak:508536
used_memory_peak_human:496.62K
used_memory_lua:33792
mem_fragmentation_ratio:15.68
mem_allocator:jemalloc-3.2.0

# Persistence
loading:0
rdb_changes_since_last_save:6
rdb_bgsave_in_progress:0
rdb_last_save_time:1411011131
rdb_last_bgsave_status:ok
rdb_last_bgsave_time_sec:-1
rdb_current_bgsave_time_sec:-1
aof_enabled:0
aof_rewrite_in_progress:0
aof_rewrite_scheduled:0
aof_last_rewrite_time_sec:-1
aof_current_rewrite_time_sec:-1
aof_last_bgrewrite_status:ok
aof_last_write_status:ok

# Stats
total_connections_received:2
total_commands_processed:4
instantaneous_ops_per_sec:0
rejected_connections:0
sync_full:0
sync_partial_ok:0
sync_partial_err:0
expired_keys:0
evicted_keys:0
keyspace_hits:0
keyspace_misses:0
pubsub_channels:0
pubsub_patterns:0
latest_fork_usec:0
migrate_cached_sockets:0

# Replication
role:master
connected_slaves:0
master_repl_offset:0
repl_backlog_active:0
repl_backlog_size:1048576
repl_backlog_first_byte_offset:0
repl_backlog_histlen:0

# CPU
used_cpu_sys:0.21
used_cpu_user:0.17
used_cpu_sys_children:0.00
used_cpu_user_children:0.00

# Cluster
cluster_enabled:0

# Keyspace
db0:keys=2,expires=0,avg_ttl=0
```

### SHUTDOWN

　　**SHUTDOWN [SAVE|NOSAVE]**

> 可用版本： >= 1.0.0
> 时间复杂度： O(N)，其中 N 为关机时需要保存的数据库键数量。

　　[SHUTDOWN](http://redisdoc.com/client_and_server/shutdown.html#shutdown) 命令执行以下操作：

- 停止所有客户端
- 如果有至少一个保存点在等待，执行 [SAVE](http://redisdoc.com/persistence/save.html#save) 命令
- 如果 AOF 选项被打开，更新 AOF 文件
- 关闭 redis 服务器(server)

　　如果持久化被打开的话， [SHUTDOWN](http://redisdoc.com/client_and_server/shutdown.html#shutdown) 命令会保证服务器正常关闭而不丢失任何数据。

　　另一方面，假如只是单纯地执行 [SAVE](http://redisdoc.com/persistence/save.html#save) 命令，然后再执行 [QUIT](http://redisdoc.com/client_and_server/quit.html#quit) 命令，则没有这一保证 —— 因为在执行 [SAVE](http://redisdoc.com/persistence/save.html#save) 之后、执行 [QUIT](http://redisdoc.com/client_and_server/quit.html#quit) 之前的这段时间中间，其他客户端可能正在和服务器进行通讯，这时如果执行 [QUIT](http://redisdoc.com/client_and_server/quit.html#quit) 就会造成数据丢失。

　　**SAVE 和 NOSAVE 修饰符**

　　通过使用可选的修饰符，可以修改 [SHUTDOWN](http://redisdoc.com/client_and_server/shutdown.html#shutdown) 命令的表现。比如说：

- 执行 `SHUTDOWN SAVE` 会强制让数据库执行保存操作，即使没有设定(configure)保存点
- 执行 `SHUTDOWN NOSAVE` 会阻止数据库执行保存操作，即使已经设定有一个或多个保存点(你可以将这一用法看作是强制停止服务器的一个假想的 ABORT 命令)

　　**返回值**

　　执行失败时返回错误。 执行成功时不返回任何信息，服务器和客户端的连接断开，客户端自动退出。

　　**代码示例**

```
redis> PING
PONG

redis> SHUTDOWN

$

$ redis
Could not connect to Redis at: Connection refused
not connected>
```

### TIME

> 可用版本： >= 2.6.0
> 时间复杂度： O(1)

　　返回当前服务器时间。

　　**返回值**

　　一个包含两个字符串的列表： 第一个字符串是当前时间(以 UNIX 时间戳格式表示)，而第二个字符串是当前这一秒钟已经逝去的微秒数。

　　**代码示例**

```
redis> TIME
1) "1332395997"
2) "952581"

redis> TIME
1) "1332395997"
2) "953148"
```

### CLIENT GETNAME

　　**CLIENT GETNAME**

　　返回 `client_setname` 命令为连接设置的名字。

　　因为新创建的连接默认是没有名字的，

　　对于没有名字的连接，

　　`client_getname` 返回空白回复。

　　**可用版本** >= 2.6.9

　　**时间复杂度**

　　O(1)

　　**返回值**

　　| 如果连接没有设置名字，那么返回空白回复；

　　| 如果有设置名字，那么返回名字。

```
# 新连接默认没有名字

redis 127.0.0.1:6379> CLIENT GETNAME
(nil)

# 设置名字

redis 127.0.0.1:6379> CLIENT SETNAME hello-world-connection
OK

# 返回名字

redis 127.0.0.1:6379> CLIENT GETNAME
"hello-world-connection"
```

### CLIENT KILL

　　**CLIENT KILL ip:port**

　　关闭地址为 `ip:port` 的客户端。

　　`ip:port` 应该和 `client_list` 命令输出的其中一行匹配。

　　因为 Redis 使用单线程设计，所以当 Redis 正在执行命令的时候，不会有客户端被断开连接。

　　如果要被断开连接的客户端正在执行命令，那么当这个命令执行之后，在发送下一个命令的时候，它就会收到一个网络错误，告知它自身的连接已被关闭。

　　**可用版本** >= 2.4.0

　　**时间复杂度**

　　O(N) ， N 为已连接的客户端数量。

　　**返回值**

　　当指定的客户端存在，且被成功关闭时，返回 `OK` 。

```
# 列出所有已连接客户端

redis 127.0.0.1:6379> CLIENT LIST
addr=127.0.0.1:43501 fd=5 age=10 idle=0 flags=N db=0 sub=0 psub=0 multi=-1 qbuf=0 qbuf-free=32768 obl=0 oll=0 omem=0 events=r cmd=client

# 杀死当前客户端的连接

redis 127.0.0.1:6379> CLIENT KILL 127.0.0.1:43501
OK

# 之前的连接已经被关闭，CLI 客户端又重新建立了连接
# 之前的端口是 43501 ，现在是 43504

redis 127.0.0.1:6379> CLIENT LIST
addr=127.0.0.1:43504 fd=5 age=0 idle=0 flags=N db=0 sub=0 psub=0 multi=-1 qbuf=0 qbuf-free=32768 obl=0 oll=0 omem=0 events=r cmd=client
```

### CLIENT LIST

　　**CLIENT LIST**

　　以人类可读的格式，返回所有连接到服务器的客户端信息和统计数据。

```
redis> CLIENT LIST
addr=127.0.0.1:43143 fd=6 age=183 idle=0 flags=N db=0 sub=0 psub=0 multi=-1 qbuf=0 qbuf-free=32768 obl=0 oll=0 omem=0 events=r cmd=client
addr=127.0.0.1:43163 fd=5 age=35 idle=15 flags=N db=0 sub=0 psub=0 multi=-1 qbuf=0 qbuf-free=0 obl=0 oll=0 omem=0 events=r cmd=ping
addr=127.0.0.1:43167 fd=7 age=24 idle=6 flags=N db=0 sub=0 psub=0 multi=-1 qbuf=0 qbuf-free=0 obl=0 oll=0 omem=0 events=r cmd=get
```

　　**可用版本** >= 2.4.0

　　**时间复杂度**

　　O(N) ， N 为连接到服务器的客户端数量。

　　**返回值**

　　命令返回多行字符串，这些字符串按以下形式被格式化：

```
* 每个已连接客户端对应一行（以 `LF` 分割）
* 每行字符串由一系列 `属性=值` 形式的域组成，每个域之间以空格分开

以下是域的含义：

* `addr` ： 客户端的地址和端口
* `fd` ： 套接字所使用的文件描述符
* `age` ： 以秒计算的已连接时长
* `idle` ： 以秒计算的空闲时长
* `flags` ： 客户端 flag （见下文）
* `db` ： 该客户端正在使用的数据库 ID
* `sub` ： 已订阅频道的数量
* `psub` ： 已订阅模式的数量
* `multi` ： 在事务中被执行的命令数量
* `qbuf` ： 查询缓冲区的长度（字节为单位， `0` 表示没有分配查询缓冲区）
* `qbuf-free` ： 查询缓冲区剩余空间的长度（字节为单位， `0` 表示没有剩余空间）
* `obl` ： 输出缓冲区的长度（字节为单位， `0` 表示没有分配输出缓冲区）
* `oll` ： 输出列表包含的对象数量（当输出缓冲区没有剩余空间时，命令回复会以字符串对象的形式被入队到这个队列里）
* `omem` ： 输出缓冲区和输出列表占用的内存总量
* `events` ： 文件描述符事件（见下文）
* `cmd` ： 最近一次执行的命令

客户端 flag 可以由以下部分组成：

* `O` ： 客户端是 MONITOR 模式下的附属节点（slave）
* `S` ： 客户端是一般模式下（normal）的附属节点
* `M` ： 客户端是主节点（master）
* `x` ： 客户端正在执行事务
* `b` ： 客户端正在等待阻塞事件
* `i` ： 客户端正在等待 VM I/O 操作（已废弃）
* `d` ： 一个受监视（watched）的键已被修改， `EXEC` 命令将失败
* `c` : 在将回复完整地写出之后，关闭链接
* `u` : 客户端未被阻塞（unblocked）
* `A` : 尽可能快地关闭连接
* `N` : 未设置任何 flag

文件描述符事件可以是：

* `r` : 客户端套接字（在事件 loop 中）是可读的（readable）
* `w` : 客户端套接字（在事件 loop 中）是可写的（writeable）
```

　　为了 debug 的需要，经常会对域进行添加和删除，一个安全的 Redis 客户端应该可以对 `CLIENT LIST` 的输出进行相应的处理（parse），比如忽略不存在的域，跳过未知域，诸如此类。

### CLIENT SETNAME

　　**CLIENT SETNAME connection-name**

　　为当前连接分配一个名字。

　　这个名字会显示在 `client_list` 命令的结果中，

　　用于识别当前正在与服务器进行连接的客户端。

　　举个例子，

　　在使用 Redis 构建队列（queue）时，

　　可以根据连接负责的任务（role），

　　为信息生产者（producer）和信息消费者（consumer）分别设置不同的名字。

　　名字使用 Redis 的字符串类型来保存，

　　最大可以占用 512 MB 。

　　另外，

　　为了避免和 `client_list` 命令的输出格式发生冲突，

　　名字里不允许使用空格。

　　要移除一个连接的名字，

　　可以将连接的名字设为空字符串 `""` 。

　　使用 `client_getname` 命令可以取出连接的名字。

　　新创建的连接默认是没有名字的。

```
在 Redis 应用程序发生连接泄漏时，为连接设置名字是一种很好的 debug 手段。
```

　　**可用版本** >= 2.6.9

　　**时间复杂度**

　　O(1)

　　**返回值**

　　设置成功时返回 `OK` 。

```
# 新连接默认没有名字

redis 127.0.0.1:6379> CLIENT GETNAME
(nil)

# 设置名字

redis 127.0.0.1:6379> CLIENT SETNAME hello-world-connection
OK

# 返回名字

redis 127.0.0.1:6379> CLIENT GETNAME
"hello-world-connection"

# 在客户端列表中查看

redis 127.0.0.1:6379> CLIENT LIST
addr=127.0.0.1:36851
fd=5
name=hello-world-connection     # <- 名字
age=51
...

# 清除名字

redis 127.0.0.1:6379> CLIENT SETNAME        # 只用空格是不行的！
(error) ERR Syntax error, try CLIENT (LIST | KILL ip:port)

redis 127.0.0.1:6379> CLIENT SETNAME ""     # 必须双引号显示包围
OK

redis 127.0.0.1:6379> CLIENT GETNAME        # 清除完毕
(nil)
```

## 配置选项

### CONFIG GET

　　**CONFIG GET parameter**

　　`CONFIG GET` 命令用于取得运行中的 Redis 服务器的配置参数(configuration parameters)，在 Redis 2.4 版本中， 有部分参数没有办法用 `CONFIG GET` 访问，但是在最新的 Redis 2.6 版本中，所有配置参数都已经可以用 `CONFIG GET` 访问了。

　　`CONFIG GET` 接受单个参数 `parameter` 作为搜索关键字，查找所有匹配的配置参数，其中参数和值以“键-值对”(key-value pairs)的方式排列。

　　比如执行 `CONFIG GET s*` 命令，服务器就会返回所有以 `s` 开头的配置参数及参数的值：

```
redis> CONFIG GET s*
1) "save"                       # 参数名：save
2) "900 1 300 10 60 10000"      # save 参数的值
3) "slave-serve-stale-data"     # 参数名： slave-serve-stale-data
4) "yes"                        # slave-serve-stale-data 参数的值
5) "set-max-intset-entries"     # ...
6) "512"
7) "slowlog-log-slower-than"
8) "1000"
9) "slowlog-max-len"
10) "1000"
```

　　如果你只是寻找特定的某个参数的话，你当然也可以直接指定参数的名字：

```
redis> CONFIG GET slowlog-max-len
1) "slowlog-max-len"
2) "1000"
```

　　使用命令 `CONFIG GET *` ，可以列出 `CONFIG GET` 命令支持的所有参数：

```
redis> CONFIG GET *
1) "dir"
2) "/var/lib/redis"
3) "dbfilename"
4) "dump.rdb"
5) "requirepass"
6) (nil)
7) "masterauth"
8) (nil)
9) "maxmemory"
10) "0"
11) "maxmemory-policy"
12) "volatile-lru"
13) "maxmemory-samples"
14) "3"
15) "timeout"
16) "0"
17) "appendonly"
18) "no"
# ...
49) "loglevel"
50) "verbose"
```

　　所有被 `CONFIG SET` 所支持的配置参数都可以在配置文件 redis.conf 中找到，不过 `CONFIG GET` 和 `CONFIG SET` 使用的格式和 redis.conf 文件所使用的格式有以下两点不同：

- | `10kb` 、 `2gb` 这些在配置文件中所使用的储存单位缩写，不可以用在 `CONFIG` 命令中， `CONFIG SET` 的值只能通过数字值显式地设定。

　　|

　　| 像 `CONFIG SET xxx 1k` 这样的命令是错误的，正确的格式是 `CONFIG SET xxx 1000` 。

- | `save` 选项在 redis.conf 中是用多行文字储存的，但在 `CONFIG GET` 命令中，它只打印一行文字。

　　|

　　| 以下是 `save` 选项在 redis.conf 文件中的表示：

　　|

　　| `save 900 1`

　　| `save 300 10`

　　| `save 60 10000`

　　|

　　| 但是 `CONFIG GET` 命令的输出只有一行：

　　|

　　| `redis> CONFIG GET save`

　　| `1) "save"`

　　| `2) "900 1 300 10 60 10000"`

　　|

　　| 上面 `save` 参数的三个值表示：在 900 秒内最少有 1 个 key 被改动，或者 300 秒内最少有 10 个 key 被改动，又或者 60 秒内最少有 1000 个 key 被改动，以上三个条件随便满足一个，就触发一次保存操作。

　　**可用版本：** >= 2.0.0

　　**时间复杂度：**

　　不明确

　　**返回值：**

　　给定配置参数的值。

### CONFIG RESETSTAT

　　**CONFIG RESETSTAT**

　　重置 `INFO` 命令中的某些统计数据，包括：

- Keyspace hits (键空间命中次数)
- Keyspace misses (键空间不命中次数)
- Number of commands processed (执行命令的次数)
- Number of connections received (连接服务器的次数)
- Number of expired keys (过期 key 的数量)
- Number of rejected connections (被拒绝的连接数量)
- Latest fork(2) time(最后执行 fork(2) 的时间)
- The `aof_delayed_fsync` counter(`aof_delayed_fsync` 计数器的值)

　　**可用版本：** >= 2.0.0

　　**时间复杂度：**

　　O(1)

　　**返回值：**

　　总是返回 `OK` 。

```
# 重置前

redis 127.0.0.1:6379> INFO
# Server
redis_version:2.5.3
redis_git_sha1:d0407c2d
redis_git_dirty:0
arch_bits:32
multiplexing_api:epoll
gcc_version:4.6.3
process_id:11095
run_id:ef1f6b6c7392e52d6001eaf777acbe547d1192e2
tcp_port:6379
uptime_in_seconds:6
uptime_in_days:0
lru_clock:1205426

# Clients
connected_clients:1
client_longest_output_list:0
client_biggest_input_buf:0
blocked_clients:0

# Memory
used_memory:331076
used_memory_human:323.32K
used_memory_rss:1568768
used_memory_peak:293424
used_memory_peak_human:286.55K
used_memory_lua:16384
mem_fragmentation_ratio:4.74
mem_allocator:jemalloc-2.2.5

# Persistence
loading:0
aof_enabled:0
changes_since_last_save:0
bgsave_in_progress:0
last_save_time:1333260015
last_bgsave_status:ok
bgrewriteaof_in_progress:0

# Stats
total_connections_received:1
total_commands_processed:0
instantaneous_ops_per_sec:0
rejected_connections:0
expired_keys:0
evicted_keys:0
keyspace_hits:0
keyspace_misses:0
pubsub_channels:0
pubsub_patterns:0
latest_fork_usec:0

# Replication
role:master
connected_slaves:0

# CPU
used_cpu_sys:0.01
used_cpu_user:0.00
used_cpu_sys_children:0.00
used_cpu_user_children:0.00

# Keyspace
db0:keys=20,expires=0


# 重置

redis 127.0.0.1:6379> CONFIG RESETSTAT
OK


# 重置后

redis 127.0.0.1:6379> INFO
# Server
redis_version:2.5.3
redis_git_sha1:d0407c2d
redis_git_dirty:0
arch_bits:32
multiplexing_api:epoll
gcc_version:4.6.3
process_id:11095
run_id:ef1f6b6c7392e52d6001eaf777acbe547d1192e2
tcp_port:6379
uptime_in_seconds:134
uptime_in_days:0
lru_clock:1205438

# Clients
connected_clients:1
client_longest_output_list:0
client_biggest_input_buf:0
blocked_clients:0

# Memory
used_memory:331076
used_memory_human:323.32K
used_memory_rss:1568768
used_memory_peak:330280
used_memory_peak_human:322.54K
used_memory_lua:16384
mem_fragmentation_ratio:4.74
mem_allocator:jemalloc-2.2.5

# Persistence
loading:0
aof_enabled:0
changes_since_last_save:0
bgsave_in_progress:0
last_save_time:1333260015
last_bgsave_status:ok
bgrewriteaof_in_progress:0

# Stats
total_connections_received:0
total_commands_processed:1
instantaneous_ops_per_sec:0
rejected_connections:0
expired_keys:0
evicted_keys:0
keyspace_hits:0
keyspace_misses:0
pubsub_channels:0
pubsub_patterns:0
latest_fork_usec:0

# Replication
role:master
connected_slaves:0

# CPU
used_cpu_sys:0.05
used_cpu_user:0.02
used_cpu_sys_children:0.00
used_cpu_user_children:0.00

# Keyspace
db0:keys=20,expires=0
```

### CONFIG REWRITE

　　**CONFIG REWRITE**

　　`CONFIG_REWRITE` 命令对启动 Redis 服务器时所指定的 `redis.conf` 文件进行改写：

　　因为 `CONFIG_SET` 命令可以对服务器的当前配置进行修改，

　　而修改后的配置可能和 `redis.conf` 文件中所描述的配置不一样，

　　`CONFIG_REWRITE` 的作用就是通过尽可能少的修改，

　　将服务器当前所使用的配置记录到 `redis.conf` 文件中。

　　重写会以非常保守的方式进行：

- 原有 `redis.conf` 文件的整体结构和注释会被尽可能地保留。
- 如果一个选项已经存在于原有 `redis.conf` 文件中 ，

　　那么对该选项的重写会在选项原本所在的位置（行号）上进行。

- 如果一个选项不存在于原有 `redis.conf` 文件中，

　　并且该选项被设置为默认值，

　　那么重写程序不会将这个选项添加到重写后的 `redis.conf` 文件中。

- 如果一个选项不存在于原有 `redis.conf` 文件中，

　　并且该选项被设置为非默认值，

　　那么这个选项将被添加到重写后的 `redis.conf` 文件的末尾。

- 未使用的行会被留白。

　　比如说，

　　如果你在原有 `redis.conf` 文件上设置了数个关于 `save` 选项的参数，

　　但现在你将这些 `save` 参数的一个或全部都关闭了，

　　那么这些不再使用的参数原本所在的行就会变成空白的。

　　即使启动服务器时所指定的 `redis.conf` 文件已经不再存在，

　　`CONFIG_REWRITE` 命令也可以重新构建并生成出一个新的 `redis.conf` 文件。

　　另一方面，

　　如果启动服务器时没有载入 `redis.conf` 文件，

　　那么执行 `CONFIG_REWRITE` 命令将引发一个错误。

#### 原子性重写

　　对 `redis.conf` 文件的重写是原子性的，

　　并且是一致的：

　　如果重写出错或重写期间服务器崩溃，

　　那么重写失败，

　　原有 `redis.conf` 文件不会被修改。

　　如果重写成功，

　　那么 `redis.conf` 文件为重写后的新文件。

#### 可用版本

> = 2.8.0

#### 返回值

　　一个状态值：如果配置重写成功则返回 `OK` ，失败则返回一个错误。

#### 测试

　　以下是执行 `CONFIG_REWRITE` 前，

　　被载入到 Redis 服务器的 `redis.conf` 文件中关于 `appendonly` 选项的设置：

```
# ... 其他选项

appendonly no

# ... 其他选项
```

　　在执行以下命令之后：

```
127.0.0.1:6379> CONFIG GET appendonly           # appendonly 处于关闭状态
1) "appendonly"
2) "no"

127.0.0.1:6379> CONFIG SET appendonly yes       # 打开 appendonly
OK

127.0.0.1:6379> CONFIG GET appendonly
1) "appendonly"
2) "yes"

127.0.0.1:6379> CONFIG REWRITE                  # 将 appendonly 的修改写入到 redis.conf 中
OK
```

　　重写后的 `redis.conf` 文件中的 `appendonly` 选项将被改写：

```
# ... 其他选项

appendonly yes

# ... 其他选项
```

### CONFIG SET

　　**CONFIG SET parameter value**

　　`CONFIG SET` 命令可以动态地调整 Redis 服务器的配置(configuration)而无须重启。

　　你可以使用它修改配置参数，或者改变 Redis 的持久化(Persistence)方式。

　　`CONFIG SET`_ 可以修改的配置参数可以使用命令 `CONFIG GET *` 来列出，所有被 `CONFIG SET`_ 修改的配置参数都会立即生效。

　　关于 `CONFIG SET` 命令的更多消息，请参见命令 `config_get` 的说明。

　　关于如何使用 `CONFIG SET`_ 命令修改 Redis 持久化方式，请参见 `Redis Persistence <http://redis.io/topics/persistence>`_ 。

　　**可用版本：** >= 2.0.0

　　**时间复杂度：**

　　不明确

　　**返回值：**

　　当设置成功时返回 `OK` ，否则返回一个错误。

```
redis> CONFIG GET slowlog-max-len
1) "slowlog-max-len"
2) "1024"

redis> CONFIG SET slowlog-max-len 10086
OK

redis> CONFIG GET slowlog-max-len
1) "slowlog-max-len"
2) "10086"
```

## 数据库

### DBSIZE

　　**DBSIZE**

　　返回当前数据库的 key 的数量。

　　**可用版本：** >= 1.0.0

　　**时间复杂度：**

　　O(1)

　　**返回值：**

　　当前数据库的 key 的数量。

```
redis> DBSIZE
(integer) 5

redis> SET new_key "hello_moto"     # 增加一个 key 试试
OK

redis> DBSIZE
(integer) 6
```

### FLUSHALL

　　**FLUSHALL**

　　清空整个 Redis 服务器的数据(删除所有数据库的所有 key )。

　　此命令从不失败。

　　**可用版本：** >= 1.0.0

　　**时间复杂度：**

　　尚未明确

　　**返回值：**

　　总是返回 `OK` 。

```
redis> DBSIZE            # 0 号数据库的 key 数量
(integer) 9

redis> SELECT 1          # 切换到 1 号数据库
OK

redis[1]> DBSIZE         # 1 号数据库的 key 数量
(integer) 6

redis[1]> flushall       # 清空所有数据库的所有 key
OK

redis[1]> DBSIZE         # 不但 1 号数据库被清空了
(integer) 0

redis[1]> SELECT 0       # 0 号数据库(以及其他所有数据库)也一样
OK

redis> DBSIZE
(integer) 0
```

### FLUSHDB

　　**FLUSHDB**

　　清空当前数据库中的所有 key。

　　此命令从不失败。

　　**可用版本：** >= 1.0.0

　　**时间复杂度：**

　　O(1)

　　**返回值：**

　　总是返回 `OK` 。

```
redis> DBSIZE    # 清空前的 key 数量
(integer) 4

redis> FLUSHDB
OK

redis> DBSIZE    # 清空后的 key 数量
(integer) 0
```

## 调试

### PING

　　**PING**

　　使用客户端向 Redis 服务器发送一个 `PING` ，如果服务器运作正常的话，会返回一个 `PONG` 。

　　通常用于测试与服务器的连接是否仍然生效，或者用于测量延迟值。

　　**可用版本：** >= 1.0.0

　　**时间复杂度：**

　　O(1)

　　**返回值：**

　　如果连接正常就返回一个 `PONG` ，否则返回一个连接错误。

```
# 客户端和服务器连接正常

redis> PING
PONG

# 客户端和服务器连接不正常(网络不正常或服务器未能正常运行)

redis 127.0.0.1:6379> PING
Could not connect to Redis at 127.0.0.1:6379: Connection refused
```

### MONITOR

　　**MONITOR**

　　实时打印出 Redis 服务器接收到的命令，调试用。

　　**可用版本：** >= 1.0.0

　　**时间复杂度：**

　　不明确

　　**返回值：**

　　总是返回 `OK` 。

```
127.0.0.1:6379> MONITOR
OK
# 以第一个打印值为例
# 1378822099.421623 是时间戳
# [0 127.0.0.1:56604] 中的 0 是数据库号码， 127... 是 IP 地址和端口
# "PING" 是被执行的命令
1378822099.421623 [0 127.0.0.1:56604] "PING"
1378822105.089572 [0 127.0.0.1:56604] "SET" "msg" "hello world"
1378822109.036925 [0 127.0.0.1:56604] "SET" "number" "123"
1378822140.649496 [0 127.0.0.1:56604] "SADD" "fruits" "Apple" "Banana" "Cherry"
1378822154.117160 [0 127.0.0.1:56604] "EXPIRE" "msg" "10086"
1378822257.329412 [0 127.0.0.1:56604] "KEYS" "*"
1378822258.690131 [0 127.0.0.1:56604] "DBSIZE"
```

### ECHO

　　**ECHO message**

　　打印一个特定的信息 `message` ，测试时使用。

　　**可用版本：** >= 1.0.0

　　**时间复杂度：**

　　O(1)

　　**返回值：**

　　`message` 自身。

```
redis> ECHO "Hello Moto"
"Hello Moto"

redis> ECHO "Goodbye Moto"
"Goodbye Moto"
```

### OBJECT

　　**OBJECT subcommand [arguments [arguments]]**

> 可用版本： >= 2.2.3
> 时间复杂度： O(1)

　　`OBJECT` 命令允许从内部察看给定 `key` 的 Redis 对象， 它通常用在除错(debugging)或者了解为了节省空间而对 `key` 使用特殊编码的情况。 当将Redis用作缓存程序时，你也可以通过 `OBJECT` 命令中的信息，决定 `key` 的驱逐策略(eviction policies)。

　　OBJECT 命令有多个子命令：

- `OBJECT REFCOUNT` 返回给定 `key` 引用所储存的值的次数。此命令主要用于除错。
- `OBJECT ENCODING` 返回给定 `key` 锁储存的值所使用的内部表示(representation)。
- `OBJECT IDLETIME` 返回给定 `key` 自储存以来的空闲时间(idle， 没有被读取也没有被写入)，以秒为单位。

　　对象可以以多种方式编码：

- 字符串可以被编码为 `raw` (一般字符串)或 `int` (为了节约内存，Redis 会将字符串表示的 64 位有符号整数编码为整数来进行储存）。
- 列表可以被编码为 `ziplist` 或 `linkedlist` 。 `ziplist` 是为节约大小较小的列表空间而作的特殊表示。
- 集合可以被编码为 `intset` 或者 `hashtable` 。 `intset` 是只储存数字的小集合的特殊表示。
- 哈希表可以编码为 `zipmap` 或者 `hashtable` 。 `zipmap` 是小哈希表的特殊表示。
- 有序集合可以被编码为 `ziplist` 或者 `skiplist` 格式。 `ziplist` 用于表示小的有序集合，而 `skiplist` 则用于表示任何大小的有序集合。

　　假如你做了什么让 Redis 没办法再使用节省空间的编码时(比如将一个只有 1 个元素的集合扩展为一个有 100 万个元素的集合)，特殊编码类型(specially encoded types)会自动转换成通用类型(general type)。

#### 返回值

　　`REFCOUNT` 和 `IDLETIME` 返回数字。 `ENCODING` 返回相应的编码类型。

#### 代码示例

```
redis> SET game "COD"           # 设置一个字符串
OK

redis> OBJECT REFCOUNT game     # 只有一个引用
(integer) 1

redis> OBJECT IDLETIME game     # 等待一阵。。。然后查看空闲时间
(integer) 90

redis> GET game                 # 提取game， 让它处于活跃(active)状态
"COD"

redis> OBJECT IDLETIME game     # 不再处于空闲状态
(integer) 0

redis> OBJECT ENCODING game     # 字符串的编码方式
"raw"

redis> SET big-number 23102930128301091820391092019203810281029831092  # 非常长的数字会被编码为字符串
OK

redis> OBJECT ENCODING big-number
"raw"

redis> SET small-number 12345  # 而短的数字则会被编码为整数
OK

redis> OBJECT ENCODING small-number
"int"
```

### SLOWLOG

　　**SLOWLOG subcommand [argument]**

　　**什么是 SLOWLOG**

　　Slow log 是 Redis 用来记录查询执行时间的日志系统。

　　查询执行时间指的是不包括像客户端响应(talking)、发送回复等 IO 操作，而单单是执行一个查询命令所耗费的时间。

　　另外，slow log 保存在内存里面，读写速度非常快，因此你可以放心地使用它，不必担心因为开启 slow log 而损害 Redis 的速度。

　　**设置 SLOWLOG**

　　Slow log 的行为由两个配置参数(configuration parameter)指定，可以通过改写 redis.conf 文件或者用 `CONFIG GET` 和 `CONFIG SET` 命令对它们动态地进行修改。

　　第一个选项是 `slowlog-log-slower-than` ，它决定要对执行时间大于多少微秒(microsecond，1 秒 = 1,000,000 微秒)的查询进行记录。

　　比如执行以下命令将让 slow log 记录所有查询时间大于等于 100 微秒的查询：

　　`CONFIG SET slowlog-log-slower-than 100`

　　而以下命令记录所有查询时间大于 1000 微秒的查询：

　　`CONFIG SET slowlog-log-slower-than 1000`

　　另一个选项是 `slowlog-max-len` ，它决定 slow log _最多_\ 能保存多少条日志， slow log 本身是一个 FIFO 队列，当队列大小超过 `slowlog-max-len` 时，最旧的一条日志将被删除，而最新的一条日志加入到 slow log ，以此类推。

　　以下命令让 slow log 最多保存 1000 条日志：

　　`CONFIG SET slowlog-max-len 1000`

　　使用 `CONFIG GET` 命令可以查询两个选项的当前值：

```
redis> CONFIG GET slowlog-log-slower-than
1) "slowlog-log-slower-than"
2) "1000"

redis> CONFIG GET slowlog-max-len
1) "slowlog-max-len"
2) "1000"
```

　　**查看 slow log**

　　要查看 slow log ，可以使用 `SLOWLOG GET` 或者 `SLOWLOG GET number` 命令，前者打印所有 slow log ，最大长度取决于 `slowlog-max-len` 选项的值，而 `SLOWLOG GET number` 则只打印指定数量的日志。

　　最新的日志会最先被打印：

```
# 为测试需要，将 slowlog-log-slower-than 设成了 10 微秒

redis> SLOWLOG GET
1) 1) (integer) 12                      # 唯一性(unique)的日志标识符
   2) (integer) 1324097834              # 被记录命令的执行时间点，以 UNIX 时间戳格式表示
   3) (integer) 16                      # 查询执行时间，以微秒为单位
   4) 1) "CONFIG"                       # 执行的命令，以数组的形式排列
      2) "GET"                          # 这里完整的命令是 CONFIG GET slowlog-log-slower-than
      3) "slowlog-log-slower-than"

2) 1) (integer) 11
   2) (integer) 1324097825
   3) (integer) 42
   4) 1) "CONFIG"
      2) "GET"
      3) "*"

3) 1) (integer) 10
   2) (integer) 1324097820
   3) (integer) 11
   4) 1) "CONFIG"
      2) "GET"
      3) "slowlog-log-slower-than"

# ...
```

　　日志的唯一 id 只有在 Redis 服务器重启的时候才会重置，这样可以避免对日志的重复处理(比如你可能会想在每次发现新的慢查询时发邮件通知你)。

　　**查看当前日志的数量**

　　使用命令 `SLOWLOG LEN` 可以查看当前日志的数量。

　　请注意这个值和 `slower-max-len` 的区别，它们一个是当前日志的数量，一个是允许记录的最大日志的数量。

```
redis> SLOWLOG LEN
(integer) 14
```

　　**清空日志**

　　使用命令 `SLOWLOG RESET` 可以清空 slow log 。

```
redis> SLOWLOG LEN
(integer) 14

redis> SLOWLOG RESET
OK

redis> SLOWLOG LEN
(integer) 0
```

　　**可用版本：** >= 2.2.12

　　**时间复杂度：**

　　O(1)

　　**返回值：**

　　取决于不同命令，返回不同的值。

### DEBUG OBJECT

　　**DEBUG OBJECT key**

　　`DEBUG OBJECT` 是一个调试命令，它不应被客户端所使用。

　　查看 `OBJECT` 命令获取更多信息。

　　**可用版本：** >= 1.0.0

　　**时间复杂度：**

　　O(1)

　　**返回值：**

　　| 当 `key` 存在时，返回有关信息。

　　| 当 `key` 不存在时，返回一个错误。

```
redis> DEBUG OBJECT my_pc
Value at:0xb6838d20 refcount:1 encoding:raw serializedlength:9 lru:283790 lru_seconds_idle:150

redis> DEBUG OBJECT your_mac
(error) ERR no such key
```

### DEBUG SEGFAULT

　　**DEBUG SEGFAULT**

　　执行一个不合法的内存访问从而让 Redis 崩溃，仅在开发时用于 BUG 模拟。

　　**可用版本：** >= 1.0.0

　　**时间复杂度：**

　　不明确

　　**返回值：**

　　无

```
redis> DEBUG SEGFAULT
Could not connect to Redis at: Connection refused

not connected>
```

## HyperLogLog

### PFADD

　　**PFADD key element [element ...]**

　　Adds all the element arguments to the HyperLogLog data structure

　　stored at the variable name specified as first argument.

　　将任意数量的元素添加到指定的 HyperLogLog 里面。

　　As a side effect of this command

　　the HyperLogLog internals may be updated

　　to reflect a different estimation of the number of unique items added so far

　　(the cardinality of the set).

　　作为这个命令的副作用，

　　HyperLogLog 内部可能会被更新，

　　以便反映一个不同的唯一元素估计数量（也即是集合的基数）。

　　If the approximated cardinality estimated by the HyperLogLog changed after executing the command,

　　PFADD returns 1,

　　otherwise 0 is returned.

　　The command automatically creates an empty HyperLogLog structure

　　(that is, a Redis String of a specified length and with a given encoding)

　　if the specified key does not exist.

　　如果 HyperLogLog 估计的近似基数（approximated cardinality）在命令执行之后出现了变化，

　　那么命令返回 `1` ，

　　否则返回 `0` 。

　　如果命令执行时给定的键不存在，

　　那么程序将先创建一个空的 HyperLogLog 结构，

　　然后再执行命令。

　　To call the command without elements but just the variable name is valid,

　　this will result into no operation performed if the variable already exists,

　　or just the creation of the data structure if the key does not exist

　　(in the latter case 1 is returned).

　　调用 `PFADD` 命令时可以只给定键名而不给定元素：

- 如果给定键已经是一个 HyperLogLog ，

　　那么这种调用不会产生任何效果；

- 但如果给定的键不存在，

　　那么命令会创建一个空的 HyperLogLog ，

　　并向客户端返回 `1` 。

　　For an introduction to HyperLogLog data structure check the PFCOUNT command page.

　　要了解更多关于 HyperLogLog 数据结构的介绍知识，

　　请查阅 `PFCOUNT` 命令的文档。

　　**可用版本：** >= 2.8.9

　　**时间复杂度：**

　　每添加一个元素的复杂度为 O(1) 。

　　**返回值：**

　　整数回复：

　　如果 HyperLogLog 的内部储存被修改了，

　　那么返回 1 ，

　　否则返回 0 。

　　Integer reply, specifically:

　　1 if at least 1 HyperLogLog internal register was altered. 0 otherwise.

```
redis> PFADD  databases  "Redis"  "MongoDB"  "MySQL"
(integer) 1

redis> PFCOUNT  databases
(integer) 3

redis> PFADD  databases  "Redis"    # Redis 已经存在，不必对估计数量进行更新
(integer) 0

redis> PFCOUNT  databases    # 元素估计数量没有变化
(integer) 3

redis> PFADD  databases  "PostgreSQL"    # 添加一个不存在的元素
(integer) 1

redis> PFCOUNT  databases    # 估计数量增一
4
```

### PFCOUNT

　　**PFCOUNT key [key ...]**

　　When call with a single key,

　　returns the approximated cardinality

　　computed by the HyperLogLog data structure stored at the specified variable,

　　which is 0 if the variable does not exist.

　　当 `PFCOUNT` 命令作用于单个键时，

　　返回储存在给定键的 HyperLogLog 的近似基数，

　　如果键不存在，

　　那么返回 `0` 。

　　When called with multiple keys,

　　returns the approximated cardinality of the union of the HyperLogLogs passed,

　　by internally merging the HyperLogLogs stored at the provided keys into a temporary hyperLogLog.

　　当 `PFCOUNT` 命令作用于多个键时，

　　返回所有给定 HyperLogLog 的并集的近似基数，

　　这个近似基数是通过将所有给定 HyperLogLog 合并至一个临时 HyperLogLog 来计算得出的。

　　The HyperLogLog data structure can be used in order to count unique elements in a set

　　using just a small constant amount of memory,

　　specifically 12k bytes for every HyperLogLog

　　(plus a few bytes for the key itself).

　　通过 HyperLogLog 数据结构，

　　用户可以使用少量固定大小的内存，

　　来储存集合中的唯一元素

　　（每个 HyperLogLog 只需使用 12k 字节内存，以及几个字节的内存来储存键本身）。

　　The returned cardinality of the observed set is not exact,

　　but approximated with a standard error of 0.81%.

　　命令返回的可见集合（observed set）基数并不是精确值，

　　而是一个带有 0.81% 标准错误（standard error）的近似值。

　　For example

　　in order to take the count of all the unique search queries performed in a day,

　　a program needs to call PFADD every time a query is processed.

　　The estimated number of unique queries can be retrieved with PFCOUNT at any time.

　　举个例子，

　　为了记录一天会执行多少次各不相同的搜索查询，

　　一个程序可以在每次执行搜索查询时调用一次 `PFADD` ，

　　并通过调用 `PFCOUNT` 命令来获取这个记录的近似结果。

　　Note: as a side effect of calling this function,

　　it is possible that the HyperLogLog is modified,

　　since the last 8 bytes encode the latest computed cardinality for caching purposes.

　　So PFCOUNT is technically a write command.

　　**可用版本：** >= 2.8.9

　　**时间复杂度：**

　　当命令作用于单个 HyperLogLog 时，

　　复杂度为 O(1) ，

　　并且具有非常低的平均常数时间。

　　当命令作用于 N 个 HyperLogLog 时，

　　复杂度为 O(N) ，

　　常数时间也比处理单个 HyperLogLog 时要大得多。

　　O(1) with every small average constant times when called with a single key.

　　O(N) with N being the number of keys,

　　and much bigger constant times, when called with multiple keys.

　　**返回值：**

　　整数回复：

　　给定 HyperLogLog 包含的唯一元素的近似数量。

　　通过 `PFADD` 命令记录的唯一元素的近似数量。

　　Integer reply, specifically:

　　The approximated number of unique elements observed via PFADD.

```
redis> PFADD  databases  "Redis"  "MongoDB"  "MySQL"
(integer) 1

redis> PFCOUNT  databases
(integer) 3

redis> PFADD  databases  "Redis"    # Redis 已经存在，不必对估计数量进行更新
(integer) 0

redis> PFCOUNT  databases    # 元素估计数量没有变化
(integer) 3

redis> PFADD  databases  "PostgreSQL"    # 添加一个不存在的元素
(integer) 1

redis> PFCOUNT  databases    # 估计数量增一
4
```

### PFMERGE

　　**PFMERGE destkey sourcekey [sourcekey ...]**

　　Merge multiple HyperLogLog values into an unique value

　　that will approximate the cardinality of the union of the observed Sets of the source HyperLogLog structures.

　　将多个 HyperLogLog 合并（merge）为一个 HyperLogLog ，

　　合并后的 HyperLogLog 的基数接近于所有输入 HyperLogLog 的可见集合（observed set）的并集。

　　The computed merged HyperLogLog is set to the destination variable,

　　which is created if does not exist (defauling to an empty HyperLogLog).

　　合并得出的 HyperLogLog 会被储存在 `destkey` 键里面，

　　如果该键并不存在，

　　那么命令在执行之前，

　　会先为该键创建一个空的 HyperLogLog 。

　　**可用版本：** >= 2.8.9

　　**时间复杂度：**

　　O(N) ，

　　其中 N 为被合并的 HyperLogLog 数量，

　　不过这个命令的常数复杂度比较高。

　　O(N) to merge N HyperLogLogs, but with high constant times.

　　**返回值：**

　　字符串回复：返回 `OK` 。

　　Simple string reply: The command just returns OK.

```
redis> PFADD  nosql  "Redis"  "MongoDB"  "Memcached"
(integer) 1

redis> PFADD  RDBMS  "MySQL" "MSSQL" "PostgreSQL"
(integer) 1

redis> PFMERGE  databases  nosql  RDBMS
OK

redis> PFCOUNT  databases
(integer) 6
```

## 发布/订阅

### PSUBSCRIBE

　　**PSUBSCRIBE pattern [pattern ...]**

　　订阅一个或多个符合给定模式的频道。

　　每个模式以 `*` 作为匹配符，比如 `it*` 匹配所有以 `it` 开头的频道( `it.news` 、 `it.blog` 、 `it.tweets` 等等)， `news.*` 匹配所有以 `news.` 开头的频道( `news.it` 、 `news.global.today` 等等)，诸如此类。

　　**可用版本：** >= 2.0.0

　　**时间复杂度：**

　　O(N)， `N` 是订阅的模式的数量。

　　**返回值：**

　　接收到的信息(请参见下面的代码说明)。

```
# 订阅 news.* 和 tweet.* 两个模式

# 第 1 - 6 行是执行 psubscribe 之后的反馈信息
# 第 7 - 10 才是接收到的第一条信息
# 第 11 - 14 是第二条
# 以此类推。。。

redis> psubscribe news.* tweet.*
Reading messages... (press Ctrl-C to quit)
1) "psubscribe"                  # 返回值的类型：显示订阅成功
2) "news.*"                      # 订阅的模式
3) (integer) 1                   # 目前已订阅的模式的数量

4) "psubscribe"
5) "tweet.*"
6) (integer) 2

7) "pmessage"                    # 返回值的类型：信息
8) "news.*"                      # 信息匹配的模式
9) "news.it"                     # 信息本身的目标频道
10) "Google buy Motorola"         # 信息的内容

11) "pmessage"
12) "tweet.*"
13) "tweet.huangz"
14) "hello"

15) "pmessage"
16) "tweet.*"
17) "tweet.joe"
18) "@huangz morning"

19) "pmessage"
20) "news.*"
21) "news.life"
22) "An apple a day, keep doctors away"
```

### PUBLISH

　　**PUBLISH channel message**

　　将信息 `message` 发送到指定的频道 `channel` 。

　　**可用版本：** >= 2.0.0

　　**时间复杂度：**

　　O(N+M)，其中 `N` 是频道 `channel` 的订阅者数量，而 `M` 则是使用模式订阅(subscribed patterns)的客户端的数量。

　　**返回值：**

　　接收到信息 `message` 的订阅者数量。

```
# 对没有订阅者的频道发送信息

redis> publish bad_channel "can any body hear me?"
(integer) 0


# 向有一个订阅者的频道发送信息

redis> publish msg "good morning"
(integer) 1


# 向有多个订阅者的频道发送信息

redis> publish chat_room "hello~ everyone"
(integer) 3
```

### PUBSUB

　　**PUBSUB ​**​**`<subcommand>`**​**​ [argument [argument ...]]**

　　`PUBSUB` 是一个查看订阅与发布系统状态的内省命令，

　　它由数个不同格式的子命令组成，

　　以下将分别对这些子命令进行介绍。

　　**可用版本：** >= 2.8.0

　　PUBSUB CHANNELS [pattern]

　　列出当前的活跃频道。

　　活跃频道指的是那些至少有一个订阅者的频道，

　　订阅模式的客户端不计算在内。

　　`pattern` 参数是可选的：

- 如果不给出 `pattern` 参数，那么列出订阅与发布系统中的所有活跃频道。
- 如果给出 `pattern` 参数，那么只列出和给定模式 `pattern` 相匹配的那些活跃频道。

　　**复杂度：** O(N) ， `N` 为活跃频道的数量（对于长度较短的频道和模式来说，将进行模式匹配的复杂度视为常数）。

　　**返回值：** 一个由活跃频道组成的列表。

```
# client-1 订阅 news.it 和 news.sport 两个频道

client-1> SUBSCRIBE news.it news.sport
Reading messages... (press Ctrl-C to quit)
1) "subscribe"
2) "news.it"
3) (integer) 1
4) "subscribe"
5) "news.sport"
6) (integer) 2

# client-2 订阅 news.it 和 news.internet 两个频道

client-2> SUBSCRIBE news.it news.internet
Reading messages... (press Ctrl-C to quit)
1) "subscribe"
2) "news.it"
3) (integer) 1
4) "subscribe"
5) "news.internet"
6) (integer) 2

# 首先， client-3 打印所有活跃频道
# 注意，即使一个频道有多个订阅者，它也只输出一次，比如 news.it

client-3> PUBSUB CHANNELS
1) "news.sport"
2) "news.internet"
3) "news.it"

# 接下来， client-3 打印那些与模式 news.i* 相匹配的活跃频道
# 因为 news.sport 不匹配 news.i* ，所以它没有被打印

redis> PUBSUB CHANNELS news.i*
1) "news.internet"
2) "news.it"
```

　　PUBSUB NUMSUB [channel-1 ... channel-N]

　　返回给定频道的订阅者数量，

　　订阅模式的客户端不计算在内。

　　**复杂度：** O(N) ， `N` 为给定频道的数量。

　　**返回值：**

　　一个多条批量回复（Multi-bulk reply），回复中包含给定的频道，以及频道的订阅者数量。

　　格式为：频道 `channel-1` ， `channel-1` 的订阅者数量，频道 `channel-2` ， `channel-2` 的订阅者数量，诸如此类。

　　回复中频道的排列顺序和执行命令时给定频道的排列顺序一致。

　　不给定任何频道而直接调用这个命令也是可以的，

　　在这种情况下，

　　命令只返回一个空列表。

```
# client-1 订阅 news.it 和 news.sport 两个频道

client-1> SUBSCRIBE news.it news.sport
Reading messages... (press Ctrl-C to quit)
1) "subscribe"
2) "news.it"
3) (integer) 1
4) "subscribe"
5) "news.sport"
6) (integer) 2

# client-2 订阅 news.it 和 news.internet 两个频道

client-2> SUBSCRIBE news.it news.internet
Reading messages... (press Ctrl-C to quit)
1) "subscribe"
2) "news.it"
3) (integer) 1
4) "subscribe"
5) "news.internet"
6) (integer) 2

# client-3 打印各个频道的订阅者数量

client-3> PUBSUB NUMSUB news.it news.internet news.sport news.music
1) "news.it"    # 频道
2) "2"          # 订阅该频道的客户端数量
3) "news.internet"
4) "1"
5) "news.sport"
6) "1"
7) "news.music" # 没有任何订阅者
8) "0"
```

　　PUBSUB NUMPAT

　　返回订阅模式的数量。

　　注意，

　　这个命令返回的不是订阅模式的客户端的数量，

　　而是客户端订阅的所有模式的数量总和。

　　**复杂度：** O(1) 。

　　**返回值：** 一个整数回复（Integer reply）。

```
# client-1 订阅 news.* 和 discount.* 两个模式

client-1> PSUBSCRIBE news.* discount.*
Reading messages... (press Ctrl-C to quit)
1) "psubscribe"
2) "news.*"
3) (integer) 1
4) "psubscribe"
5) "discount.*"
6) (integer) 2

# client-2 订阅 tweet.* 一个模式

client-2> PSUBSCRIBE tweet.*
Reading messages... (press Ctrl-C to quit)
1) "psubscribe"
2) "tweet.*"
3) (integer) 1

# client-3 返回当前订阅模式的数量为 3

client-3> PUBSUB NUMPAT
(integer) 3

# 注意，当有多个客户端订阅相同的模式时，相同的订阅也被计算在 PUBSUB NUMPAT 之内
# 比如说，再新建一个客户端 client-4 ，让它也订阅 news.* 频道

client-4> PSUBSCRIBE news.*
Reading messages... (press Ctrl-C to quit)
1) "psubscribe"
2) "news.*"
3) (integer) 1

# 这时再计算被订阅模式的数量，就会得到数量为 4

client-3> PUBSUB NUMPAT
(integer) 4
```

### PUNSUBSCRIBE

　　**PUNSUBSCRIBE [pattern [pattern ...]]**

　　指示客户端退订所有给定模式。

　　如果没有模式被指定，也即是，一个无参数的 `PUNSUBSCRIBE` 调用被执行，那么客户端使用 `PSUBSCRIBE` 命令订阅的所有模式都会被退订。在这种情况下，命令会返回一个信息，告知客户端所有被退订的模式。

　　**可用版本：** >= 2.0.0

　　**时间复杂度：**

　　O(N+M) ，其中 `N` 是客户端已订阅的模式的数量， `M` 则是系统中所有客户端订阅的模式的数量。

　　**返回值：**

　　这个命令在不同的客户端中有不同的表现。

### SUBSCRIBE

　　**SUBSCRIBE channel [channel ...]**

　　订阅给定的一个或多个频道的信息。

　　**可用版本：** >= 2.0.0

　　**时间复杂度：**

　　O(N)，其中 `N` 是订阅的频道的数量。

　　**返回值：**

　　接收到的信息(请参见下面的代码说明)。

```
# 订阅 msg 和 chat_room 两个频道

# 1 - 6 行是执行 subscribe 之后的反馈信息
# 第 7 - 9 行才是接收到的第一条信息
# 第 10 - 12 行是第二条

redis> subscribe msg chat_room
Reading messages... (press Ctrl-C to quit)
1) "subscribe"       # 返回值的类型：显示订阅成功
2) "msg"             # 订阅的频道名字
3) (integer) 1       # 目前已订阅的频道数量

4) "subscribe"
5) "chat_room"
6) (integer) 2

7) "message"         # 返回值的类型：信息
8) "msg"             # 来源(从那个频道发送过来)
9) "hello moto"      # 信息内容

10) "message"
11) "chat_room"
12) "testing...haha"
```

### UNSUBSCRIBE

　　**UNSUBSCRIBE [channel [channel ...]]**

　　指示客户端退订给定的频道。

　　如果没有频道被指定，也即是，一个无参数的 `UNSUBSCRIBE` 调用被执行，那么客户端使用 `SUBSCRIBE` 命令订阅的所有频道都会被退订。在这种情况下，命令会返回一个信息，告知客户端所有被退订的频道。

　　**可用版本：** >= 2.0.0

　　**时间复杂度：**

　　O(N) ， `N` 是客户端已订阅的频道的数量。

　　**返回值：**

　　这个命令在不同的客户端中有不同的表现。

## 复制

### SLAVEOF

　　function SLAVEOF host port

　　`SLAVEOF`命令用于在 Redis 运行时动态地修改复制(replication)功能的行为。

　　通过执行 `SLAVEOF host port` 命令，可以将当前服务器转变为指定服务器的从属服务器(slave server)。

　　如果当前服务器已经是某个主服务器(master server)的从属服务器，那么执行 `SLAVEOF host port` 将使当前服务器停止对旧主服务器的同步，丢弃旧数据集，转而开始对新主服务器进行同步。

　　另外，对一个从属服务器执行命令 `SLAVEOF NO ONE` 将使得这个从属服务器关闭复制功能，并从从属服务器转变回主服务器，原来同步所得的数据集不会被丢弃。

　　利用『 `SLAVEOF NO ONE` 不会丢弃同步所得数据集』这个特性，可以在主服务器失败的时候，将从属服务器用作新的主服务器，从而实现无间断运行。

　　**可用版本：** >= 1.0.0

　　**时间复杂度：**

　　| `SLAVEOF host port` ，O(N)， `N` 为要同步的数据数量。

　　| `SLAVEOF NO ONE` ， O(1) 。

　　**返回值：**

　　总是返回 `OK` 。

```
redis> SLAVEOF 127.0.0.1 6379
OK

redis> SLAVEOF NO ONE
OK
```

### Role

　　**ROLE**

> 可用版本： >= 2.8.12
> 时间复杂度： O(1)

　　返回实例在复制中担任的角色， 这个角色可以是 `master` 、 `slave` 或者 `sentinel` 。 除了角色之外， 命令还会返回与该角色相关的其他信息， 其中：

- 主服务器将返回属下从服务器的 IP 地址和端口。
- 从服务器将返回自己正在复制的主服务器的 IP 地址、端口、连接状态以及复制偏移量。
- Sentinel 将返回自己正在监视的主服务器列表。

　　**返回值**

　　`ROLE` 命令将返回一个数组。

　　**代码示例**

　　**主服务器**

```
1) "master"
2) (integer) 3129659
3) 1) 1) "127.0.0.1"
      1) "9001"
      2) "3129242"
   1) 1) "127.0.0.1"
      1) "9002"
      2) "3129543"
```

　　**从服务器**

```
1) "slave"
2) "127.0.0.1"
3) (integer) 9000
4) "connected"
5) (integer) 3167038
```

　　**Sentinel**

```
1) "sentinel"
2) 1) "resque-master"
   2) "html-fragments-master"
   3) "stats-master"
   4) "metadata-master"
```

## 持久化

### SAVE

　　**SAVE**

　　`SAVE` 命令执行一个同步保存操作，将当前 Redis 实例的所有数据快照(snapshot)以 RDB 文件的形式保存到硬盘。

　　一般来说，在生产环境很少执行 `SAVE`_ 操作，因为它会阻塞所有客户端，保存数据库的任务通常由 `BGSAVE` 命令异步地执行。然而，如果负责保存数据的后台子进程不幸出现问题时， `SAVE`_ 可以作为保存数据的最后手段来使用。

　　请参考文档： `Redis 的持久化运作方式(英文) <http://redis.io/topics/persistence>` 以获取更多消息。

　　**可用版本：** >= 1.0.0

　　**时间复杂度：**

　　O(N)， `N` 为要保存到数据库中的 key 的数量。

　　**返回值：**

　　保存成功时返回 `OK` 。

```
redis> SAVE
OK
```

### BGSAVE

　　在后台异步(Asynchronously)保存当前数据库的数据到磁盘。

　　`BGSAVE` 命令执行之后立即返回 `OK` ，然后 Redis fork 出一个新子进程，原来的 Redis 进程(父进程)继续处理客户端请求，而子进程则负责将数据保存到磁盘，然后退出。

　　客户端可以通过 `LASTSAVE` 命令查看相关信息，判断 `BGSAVE` 命令是否执行成功。

　　请移步 `持久化文档 <http://redis.io/topics/persistence>` 查看更多相关细节。

　　**可用版本：** >= 1.0.0

　　**时间复杂度：**

　　O(N)， `N` 为要保存到数据库中的 key 的数量。

　　**返回值：**

　　反馈信息。

```
redis> BGSAVE
Background saving started
```

### BGREWRITEAOF

　　=============

　　**BGREWRITEAOF**

　　执行一个 `AOF文件 <http://redis.io/topics/persistence#append-only-file>` 重写操作。重写会创建一个当前 AOF 文件的体积优化版本。

　　即使 `BGREWRITEAOF`_ 执行失败，也不会有任何数据丢失，因为旧的 AOF 文件在 `BGREWRITEAOF`_ 成功之前不会被修改。

　　重写操作只会在没有其他持久化工作在后台执行时被触发，也就是说：

- 如果 Redis 的子进程正在执行快照的保存工作，那么 AOF 重写的操作会被预定(scheduled)，等到保存工作完成之后再执行 AOF 重写。在这种情况下， `BGREWRITEAOF`_ 的返回值仍然是 `OK` ，但还会加上一条额外的信息，说明 `BGREWRITEAOF`_ 要等到保存操作完成之后才能执行。在 Redis 2.6 或以上的版本，可以使用 `INFO` 命令查看 `BGREWRITEAOF` 是否被预定。
- 如果已经有别的 AOF 文件重写在执行，那么 `BGREWRITEAOF`_ 返回一个错误，并且这个新的 `BGREWRITEAOF`_ 请求也不会被预定到下次执行。

　　从 Redis 2.4 开始， AOF 重写由 Redis 自行触发， `BGREWRITEAOF` 仅仅用于手动触发重写操作。

　　请移步 `持久化文档(英文) <http://redis.io/topics/persistence>` 查看更多相关细节。

　　**可用版本：** >= 1.0.0

　　**时间复杂度：**

　　O(N)， `N` 为要追加到 AOF 文件中的数据数量。

　　**返回值：**

　　反馈信息。

```
redis> BGREWRITEAOF
Background append only file rewriting started
```

### LASTSAVE

　　=========

　　**LASTSAVE**

　　返回最近一次 Redis 成功将数据保存到磁盘上的时间，以 UNIX 时间戳格式表示。

　　**可用版本：** >= 1.0.0

　　**时间复杂度：**

　　O(1)

　　**返回值：**

　　一个 UNIX 时间戳。

```
redis> LASTSAVE
(integer) 1324043588
```

## Lua 脚本

### EVAL

　　**EVAL script numkeys key [key ...] arg [arg ...]**

　　从 Redis 2.6.0 版本开始，通过内置的 Lua 解释器，可以使用 `EVAL` 命令对 Lua 脚本进行求值。

　　`script` 参数是一段 Lua 5.1 脚本程序，它会被运行在 Redis 服务器上下文中，这段脚本不必(也不应该)定义为一个 Lua 函数。

　　`numkeys` 参数用于指定键名参数的个数。

　　键名参数 `key [key ...]` 从 `EVAL` 的第三个参数开始算起，表示在脚本中所用到的那些 Redis 键(key)，这些键名参数可以在 Lua 中通过全局变量 `KEYS` 数组，用 `1` 为基址的形式访问( `KEYS[1]` ， `KEYS[2]` ，以此类推)。

　　在命令的最后，那些不是键名参数的附加参数 `arg [arg ...]` ，可以在 Lua 中通过全局变量 `ARGV` 数组访问，访问的形式和 `KEYS` 变量类似( `ARGV[1]` 、 `ARGV[2]` ，诸如此类)。

　　上面这几段长长的说明可以用一个简单的例子来概括：

```
> eval "return {KEYS[1],KEYS[2],ARGV[1],ARGV[2]}" 2 key1 key2 first second
1) "key1"
2) "key2"
3) "first"
4) "second"
```

　　其中 `"return {KEYS[1],KEYS[2],ARGV[1],ARGV[2]}"` 是被求值的 Lua 脚本，数字 `2` 指定了键名参数的数量， `key1` 和 `key2` 是键名参数，分别使用 `KEYS[1]` 和 `KEYS[2]` 访问，而最后的 `first` 和 `second` 则是附加参数，可以通过 `ARGV[1]` 和 `ARGV[2]` 访问它们。

　　在 Lua 脚本中，可以使用两个不同函数来执行 Redis 命令，它们分别是：

- `redis.call()`
- `redis.pcall()`

　　这两个函数的唯一区别在于它们使用不同的方式处理执行命令所产生的错误，在后面的『错误处理』部分会讲到这一点。

　　`redis.call()` 和 `redis.pcall()` 两个函数的参数可以是任何格式良好(well formed)的 Redis 命令：

```
> eval "return redis.call('set','foo','bar')" 0
OK
```

　　需要注意的是，上面这段脚本的确实现了将键 `foo` 的值设为 `bar` 的目的，但是，它违反了 `EVAL` 命令的语义，因为脚本里使用的所有键都应该由 `KEYS` 数组来传递，就像这样：

```
> eval "return redis.call('set',KEYS[1],'bar')" 1 foo
OK
```

　　要求使用正确的形式来传递键(key)是有原因的，因为不仅仅是 `EVAL` 这个命令，所有的 Redis 命令，在执行之前都会被分析，籍此来确定命令会对哪些键进行操作。

　　因此，对于 `EVAL` 命令来说，必须使用正确的形式来传递键，才能确保分析工作正确地执行。除此之外，使用正确的形式来传递键还有很多其他好处，它的一个特别重要的用途就是确保 Redis 集群可以将你的请求发送到正确的集群节点。(对 Redis 集群的工作还在进行当中，但是脚本功能被设计成可以与集群功能保持兼容。)不过，这条规矩并不是强制性的，从而使得用户有机会滥用(abuse) Redis 单实例配置(single instance configuration)，代价是这样写出的脚本不能被 Redis 集群所兼容。

#### 在 Lua 数据类型和 Redis 数据类型之间转换

---

　　当 Lua 通过 `call()` 或 `pcall()` 函数执行 Redis 命令的时候，命令的返回值会被转换成 Lua 数据结构。同样地，当 Lua 脚本在 Redis 内置的解释器里运行时，Lua 脚本的返回值也会被转换成 Redis 协议(protocol)，然后由 `EVAL` 将值返回给客户端。

　　数据类型之间的转换遵循这样一个设计原则：如果将一个 Redis 值转换成 Lua 值，之后再将转换所得的 Lua 值转换回 Redis 值，那么这个转换所得的 Redis 值应该和最初时的 Redis 值一样。

　　换句话说， Lua 类型和 Redis 类型之间存在着一一对应的转换关系。

　　以下列出的是详细的转换规则：

　　从 Redis 转换到 Lua ：

- Redis integer reply -> Lua number / Redis 整数转换成 Lua 数字
- Redis bulk reply -> Lua string / Redis bulk 回复转换成 Lua 字符串
- Redis multi bulk reply -> Lua table (may have other Redis data types nested) / Redis 多条 bulk 回复转换成 Lua 表，表内可能有其他别的 Redis 数据类型
- Redis status reply -> Lua table with a single ok field containing the status / Redis 状态回复转换成 Lua 表，表内的 `ok` 域包含了状态信息
- Redis error reply -> Lua table with a single err field containing the error / Redis 错误回复转换成 Lua 表，表内的 `err` 域包含了错误信息
- Redis Nil bulk reply and Nil multi bulk reply -> Lua false boolean type / Redis 的 Nil 回复和 Nil 多条回复转换成 Lua 的布尔值 `false`

　　从 Lua 转换到 Redis：

- Lua number -> Redis integer reply / Lua 数字转换成 Redis 整数
- Lua string -> Redis bulk reply / Lua 字符串转换成 Redis bulk 回复
- Lua table (array) -> Redis multi bulk reply / Lua 表(数组)转换成 Redis 多条 bulk 回复
- Lua table with a single ok field -> Redis status reply / 一个带单个 `ok` 域的 Lua 表，转换成 Redis 状态回复
- Lua table with a single err field -> Redis error reply / 一个带单个 `err` 域的 Lua 表，转换成 Redis 错误回复
- Lua boolean false -> Redis Nil bulk reply / Lua 的布尔值 `false` 转换成 Redis 的 Nil bulk 回复

　　从 Lua 转换到 Redis 有一条额外的规则，这条规则没有和它对应的从 Redis 转换到 Lua 的规则：

- Lua boolean true -> Redis integer reply with value of 1 / Lua 布尔值 `true` 转换成 Redis 整数回复中的 `1`

　　以下是几个类型转换的例子：

```
> eval "return 10" 0
(integer) 10

> eval "return {1,2,{3,'Hello World!'}}" 0
1) (integer) 1
2) (integer) 2
3) 1) (integer) 3
   2) "Hello World!"

> eval "return redis.call('get','foo')" 0
"bar"
```

　　在上面的三个代码示例里，前两个演示了如何将 Lua 值转换成 Redis 值，最后一个例子更复杂一些，它演示了一个将 Redis 值转换成 Lua 值，然后再将 Lua 值转换成 Redis 值的类型转过程。

#### 脚本的原子性

---

　　Redis 使用单个 Lua 解释器去运行所有脚本，并且， Redis 也保证脚本会以原子性(atomic)的方式执行：当某个脚本正在运行的时候，不会有其他脚本或 Redis 命令被执行。这和使用 `MULTI` / `EXEC` 包围的事务很类似。在其他别的客户端看来，脚本的效果(effect)要么是不可见的(not visible)，要么就是已完成的(already completed)。

　　另一方面，这也意味着，执行一个运行缓慢的脚本并不是一个好主意。写一个跑得很快很顺溜的脚本并不难，因为脚本的运行开销(overhead)非常少，但是当你不得不使用一些跑得比较慢的脚本时，请小心，因为当这些蜗牛脚本在慢吞吞地运行的时候，其他客户端会因为服务器正忙而无法执行命令。

#### 错误处理

---

　　前面的命令介绍部分说过， `redis.call()` 和 `redis.pcall()` 的唯一区别在于它们对错误处理的不同。

　　当 `redis.call()` 在执行命令的过程中发生错误时，脚本会停止执行，并返回一个脚本错误，错误的输出信息会说明错误造成的原因：

```
redis> lpush foo a
(integer) 1

redis> eval "return redis.call('get', 'foo')" 0
(error) ERR Error running script (call to f_282297a0228f48cd3fc6a55de6316f31422f5d17): ERR Operation against a key holding the wrong kind of value
```

　　和 `redis.call()` 不同， `redis.pcall()` 出错时并不引发(raise)错误，而是返回一个带 `err` 域的 Lua 表(table)，用于表示错误：

```
redis 127.0.0.1:6379> EVAL "return redis.pcall('get', 'foo')" 0
(error) ERR Operation against a key holding the wrong kind of value
```

#### 带宽和 EVALSHA

---

　　`EVAL` 命令要求你在每次执行脚本的时候都发送一次脚本主体(script body)。Redis 有一个内部的缓存机制，因此它不会每次都重新编译脚本，不过在很多场合，付出无谓的带宽来传送脚本主体并不是最佳选择。

　　为了减少带宽的消耗， Redis 实现了 EVALSHA 命令，它的作用和 `EVAL` 一样，都用于对脚本求值，但它接受的第一个参数不是脚本，而是脚本的 SHA1 校验和(sum)。

　　EVALSHA 命令的表现如下：

- 如果服务器还记得给定的 SHA1 校验和所指定的脚本，那么执行这个脚本
- 如果服务器不记得给定的 SHA1 校验和所指定的脚本，那么它返回一个特殊的错误，提醒用户使用 `EVAL` 代替 EVALSHA

　　以下是示例：

```
> set foo bar
OK

> eval "return redis.call('get','foo')" 0
"bar"

> evalsha 6b1bf486c81ceb7edf3c093f4c48582e38c0e791 0
"bar"

> evalsha ffffffffffffffffffffffffffffffffffffffff 0
(error) `NOSCRIPT` No matching script. Please use [EVAL](/commands/eval).
```

　　客户端库的底层实现可以一直乐观地使用 EVALSHA 来代替 `EVAL`_ ，并期望着要使用的脚本已经保存在服务器上了，只有当 `NOSCRIPT` 错误发生时，才使用 `EVAL`_ 命令重新发送脚本，这样就可以最大限度地节省带宽。

　　这也说明了执行 `EVAL` 命令时，使用正确的格式来传递键名参数和附加参数的重要性：因为如果将参数硬写在脚本中，那么每次当参数改变的时候，都要重新发送脚本，即使脚本的主体并没有改变，相反，通过使用正确的格式来传递键名参数和附加参数，就可以在脚本主体不变的情况下，直接使用 EVALSHA 命令对脚本进行复用，免去了无谓的带宽消耗。

#### 脚本缓存

---

　　Redis 保证所有被运行过的脚本都会被永久保存在脚本缓存当中，这意味着，当 `EVAL` 命令在一个 Redis 实例上成功执行某个脚本之后，随后针对这个脚本的所有 EVALSHA 命令都会成功执行。

　　刷新脚本缓存的唯一办法是显式地调用 `SCRIPT FLUSH` 命令，这个命令会清空运行过的所有脚本的缓存。通常只有在云计算环境中，Redis 实例被改作其他客户或者别的应用程序的实例时，才会执行这个命令。

　　缓存可以长时间储存而不产生内存问题的原因是，它们的体积非常小，而且数量也非常少，即使脚本在概念上类似于实现一个新命令，即使在一个大规模的程序里有成百上千的脚本，即使这些脚本会经常修改，即便如此，储存这些脚本的内存仍然是微不足道的。

　　事实上，用户会发现 Redis 不移除缓存中的脚本实际上是一个好主意。比如说，对于一个和 Redis 保持持久化链接(persistent connection)的程序来说，它可以确信，执行过一次的脚本会一直保留在内存当中，因此它可以在流水线中使用 EVALSHA 命令而不必担心因为找不到所需的脚本而产生错误(稍候我们会看到在流水线中执行脚本的相关问题)。

#### SCRIPT 命令

---

　　Redis 提供了以下几个 SCRIPT 命令，用于对脚本子系统(scripting subsystem)进行控制：

- `script_flush` ：清除所有脚本缓存
- `script_exists` ：根据给定的脚本校验和，检查指定的脚本是否存在于脚本缓存
- `script_load` ：将一个脚本装入脚本缓存，但并不立即运行它
- `script_kill` ：杀死当前正在运行的脚本

#### 纯函数脚本

---

　　在编写脚本方面，一个重要的要求就是，脚本应该被写成纯函数(pure function)。

　　也就是说，脚本应该具有以下属性：

- 对于同样的数据集输入，给定相同的参数，脚本执行的 Redis 写命令总是相同的。脚本执行的操作不能依赖于任何隐藏(非显式)数据，不能依赖于脚本在执行过程中、或脚本在不同执行时期之间可能变更的状态，并且它也不能依赖于任何来自 I/O 设备的外部输入。

　　使用系统时间(system time)，调用像 `RANDOMKEY` 那样的随机命令，或者使用 Lua 的随机数生成器，类似以上的这些操作，都会造成脚本的求值无法每次都得出同样的结果。

　　为了确保脚本符合上面所说的属性， Redis 做了以下工作：

- Lua 没有访问系统时间或者其他内部状态的命令
- Redis 会返回一个错误，阻止这样的脚本运行： 这些脚本在执行随机命令之后(比如 `RANDOMKEY` 、 `SRANDMEMBER` 或 `TIME` 等)，还会执行可以修改数据集的 Redis 命令。如果脚本只是执行只读操作，那么就没有这一限制。注意，随机命令并不一定就指那些带 RAND 字眼的命令，任何带有非确定性的命令都会被认为是随机命令，比如 `TIME` 命令就是这方面的一个很好的例子。
- 每当从 Lua 脚本中调用那些返回无序元素的命令时，执行命令所得的数据在返回给 Lua 之前会先执行一个静默(slient)的字典序排序(`lexicographical sorting <http://en.wikipedia.org/wiki/Lexicographical_order>`)。举个例子，因为 Redis 的 Set 保存的是无序的元素，所以在 Redis 命令行客户端中直接执行 `SMEMBERS` ，返回的元素是无序的，但是，假如在脚本中执行 `redis.call("smembers", KEYS[1])` ，那么返回的总是排过序的元素。
- 对 Lua 的伪随机数生成函数 `math.random` 和 `math.randomseed` 进行修改，使得每次在运行新脚本的时候，总是拥有同样的 seed 值。这意味着，每次运行脚本时，只要不使用 `math.randomseed` ，那么 `math.random` 产生的随机数序列总是相同的。

　　尽管有那么多的限制，但用户还是可以用一个简单的技巧写出带随机行为的脚本(如果他们需要的话)。

　　假设现在我们要编写一个 Redis 脚本，这个脚本从列表中弹出 N 个随机数。一个 Ruby 写的例子如下：

```
require 'rubygems'
require 'redis'

r = Redis.new

RandomPushScript = <<EOF
    local i = tonumber(ARGV[1])
    local res
    while (i > 0) do
        res = redis.call('lpush',KEYS[1],math.random())
        i = i-1
    end
    return res
EOF

r.del(:mylist)
puts r.eval(RandomPushScript,[:mylist],[10,rand(2**32)])
```

　　这个程序每次运行都会生成带有以下元素的列表：

```
> lrange mylist 0 -1
1) "0.74509509873814"
2) "0.87390407681181"
3) "0.36876626981831"
4) "0.6921941534114"
5) "0.7857992587545"
6) "0.57730350670279"
7) "0.87046522734243"
8) "0.09637165539729"
9) "0.74990198051087"
10) "0.17082803611217"
```

　　上面的 Ruby 程序每次都只生成同样的列表，用途并不是太大。那么，该怎样修改这个脚本，使得它仍然是一个纯函数(符合 Redis 的要求)，但是每次调用都可以产生不同的随机元素呢？

　　一个简单的办法是，为脚本添加一个额外的参数，让这个参数作为 Lua 的随机数生成器的 seed 值，这样的话，只要给脚本传入不同的 seed ，脚本就会生成不同的列表元素。

　　以下是修改后的脚本：

```
RandomPushScript = <<EOF
    local i = tonumber(ARGV[1])
    local res
    math.randomseed(tonumber(ARGV[2]))
    while (i > 0) do
        res = redis.call('lpush',KEYS[1],math.random())
        i = i-1
    end
    return res
EOF

r.del(:mylist)
puts r.eval(RandomPushScript,1,:mylist,10,rand(2**32))
```

　　尽管对于同样的 seed ，上面的脚本产生的列表元素是一样的(因为它是一个纯函数)，但是只要每次在执行脚本的时候传入不同的 seed ，我们就可以得到带有不同随机元素的列表。

　　Seed 会在复制(replication link)和写 AOF 文件时作为一个参数来传播，保证在载入 AOF 文件或附属节点(slave)处理脚本时， seed 仍然可以及时得到更新。

　　注意，Redis 实现保证 `math.random` 和 `math.randomseed` 的输出和运行 Redis 的系统架构无关，无论是 32 位还是 64 位系统，无论是小端(little endian)还是大端(big endian)系统，这两个函数的输出总是相同的。

#### 全局变量保护

---

　　为了防止不必要的数据泄漏进 Lua 环境， Redis 脚本不允许创建全局变量。如果一个脚本需要在多次执行之间维持某种状态，它应该使用 Redis key 来进行状态保存。

　　企图在脚本中访问一个全局变量(不论这个变量是否存在)将引起脚本停止， `EVAL` 命令会返回一个错误：

```
redis 127.0.0.1:6379> eval 'a=10' 0
(error) ERR Error running script (call to f_933044db579a2f8fd45d8065f04a8d0249383e57): user_script:1: Script attempted to create global variable 'a'
```

　　Lua 的 debug 工具，或者其他设施，比如打印（alter）用于实现全局保护的 meta table ，都可以用于实现全局变量保护。

　　实现全局变量保护并不难，不过有时候还是会不小心而为之。一旦用户在脚本中混入了 Lua 全局状态，那么 AOF 持久化和复制（replication）都会无法保证，所以，请不要使用全局变量。

　　避免引入全局变量的一个诀窍是：将脚本中用到的所有变量都使用 `local` 关键字定义为局部变量。

#### 库

---

　　Redis 内置的 Lua 解释器加载了以下 Lua 库：

- `base`
- `table`
- `string`
- `math`
- `debug`
- `cjson`
- `cmsgpack`

　　其中 `cjson` 库可以让 Lua 以非常快的速度处理 JSON 数据，除此之外，其他别的都是 Lua 的标准库。

　　每个 Redis 实例都保证会加载上面列举的库，从而确保每个 Redis 脚本的运行环境都是相同的。

#### 使用脚本散发 Redis 日志

---

　　在 Lua 脚本中，可以通过调用 `redis.log` 函数来写 Redis 日志(log)：

　　`redis.log(loglevel, message)`

　　其中， `message` 参数是一个字符串，而 `loglevel` 参数可以是以下任意一个值：

- `redis.LOG_DEBUG`
- `redis.LOG_VERBOSE`
- `redis.LOG_NOTICE`
- `redis.LOG_WARNING`

　　上面的这些等级(level)和标准 Redis 日志的等级相对应。

　　对于脚本散发(emit)的日志，只有那些和当前 Redis 实例所设置的日志等级相同或更高级的日志才会被散发。

　　以下是一个日志示例：

　　`redis.log(redis.LOG_WARNING, "Something is wrong with this script.")`

　　执行上面的函数会产生这样的信息：

　　`[32343] 22 Mar 15:21:39 # Something is wrong with this script.`

#### 沙箱(sandbox)和最大执行时间

---

　　脚本应该仅仅用于传递参数和对 Redis 数据进行处理，它不应该尝试去访问外部系统(比如文件系统)，或者执行任何系统调用。

　　除此之外，脚本还有一个最大执行时间限制，它的默认值是 5 秒钟，一般正常运作的脚本通常可以在几分之几毫秒之内完成，花不了那么多时间，这个限制主要是为了防止因编程错误而造成的无限循环而设置的。

　　最大执行时间的长短由 `lua-time-limit` 选项来控制(以毫秒为单位)，可以通过编辑 `redis.conf` 文件或者使用 `config_get` 和 `config_set` 命令来修改它。

　　当一个脚本达到最大执行时间的时候，它并不会自动被 Redis 结束，因为 Redis 必须保证脚本执行的原子性，而中途停止脚本的运行意味着可能会留下未处理完的数据在数据集(data set)里面。

　　因此，当脚本运行的时间超过最大执行时间后，以下动作会被执行：

- Redis 记录一个脚本正在超时运行
- Redis 开始重新接受其他客户端的命令请求，但是只有 `SCRIPT KILL` 和 `SHUTDOWN NOSAVE` 两个命令会被处理，对于其他命令请求， Redis 服务器只是简单地返回 `BUSY` 错误。
- 可以使用 `SCRIPT KILL` 命令将一个仅执行只读命令的脚本杀死，因为只读命令并不修改数据，因此杀死这个脚本并不破坏数据的完整性
- 如果脚本已经执行过写命令，那么唯一允许执行的操作就是 `SHUTDOWN NOSAVE` ，它通过停止服务器来阻止当前数据集写入磁盘

#### 流水线(pipeline)上下文(context)中的 EVALSHA

---

　　在流水线请求的上下文中使用 EVALSHA 命令时，要特别小心，因为在流水线中，必须保证命令的执行顺序。

　　一旦在流水线中因为 EVALSHA 命令而发生 NOSCRIPT 错误，那么这个流水线就再也没有办法重新执行了，否则的话，命令的执行顺序就会被打乱。

　　为了防止出现以上所说的问题，客户端库实现应该实施以下的其中一项措施：

- 总是在流水线中使用 `EVAL` 命令
- 检查流水线中要用到的所有命令，找到其中的 `EVAL`_ 命令，并使用 `SCRIPT_EXISTS` 命令检查要用到的脚本是不是全都已经保存在缓存里面了。如果所需的全部脚本都可以在缓存里找到，那么就可以放心地将所有 `EVAL`_ 命令改成 EVALSHA 命令，否则的话，就要在流水线的顶端(top)将缺少的脚本用 `script_load` 命令加上去。

　　**可用版本：** >= 2.6.0

　　**时间复杂度：**

　　`EVAL` 和 EVALSHA 可以在 O(1) 复杂度内找到要被执行的脚本，其余的复杂度取决于执行的脚本本身。

### EVALSHA

　　**EVALSHA sha1 numkeys key [key ...] arg [arg ...]**

　　根据给定的 sha1 校验码，对缓存在服务器中的脚本进行求值。

　　将脚本缓存到服务器的操作可以通过 `script_load` 命令进行。

　　这个命令的其他地方，比如参数的传入方式，都和 `eval` 命令一样。

　　**可用版本：** >= 2.6.0

　　**时间复杂度：**

　　根据脚本的复杂度而定。

```
redis> SCRIPT LOAD "return 'hello moto'"
"232fd51614574cf0867b83d384a5e898cfd24e5a"

redis> EVALSHA "232fd51614574cf0867b83d384a5e898cfd24e5a" 0
"hello moto"
```

### SCRIPT EXISTS

　　**SCRIPT EXISTS script [script ...]**

　　给定一个或多个脚本的 SHA1 校验和，返回一个包含 `0` 和 `1` 的列表，表示校验和所指定的脚本是否已经被保存在缓存当中。

　　关于使用 Redis 对 Lua 脚本进行求值的更多信息，请参见 `EVAL` 命令。

　　**可用版本：** >= 2.6.0

　　**时间复杂度：**

　　O(N) , `N` 为给定的 SHA1 校验和的数量。

　　**返回值：**

　　| 一个列表，包含 `0` 和 `1` ，前者表示脚本不存在于缓存，后者表示脚本已经在缓存里面了。

　　| 列表中的元素和给定的 SHA1 校验和保持对应关系，比如列表的第三个元素的值就表示第三个 SHA1 校验和所指定的脚本在缓存中的状态。

```
redis> SCRIPT LOAD "return 'hello moto'"    # 载入一个脚本
"232fd51614574cf0867b83d384a5e898cfd24e5a"

redis> SCRIPT EXISTS 232fd51614574cf0867b83d384a5e898cfd24e5a
1) (integer) 1

redis> SCRIPT FLUSH     # 清空缓存
OK

redis> SCRIPT EXISTS 232fd51614574cf0867b83d384a5e898cfd24e5a
1) (integer) 0
```

### SCRIPT FLUSH

　　**SCRIPT FLUSH**

　　清除所有 Lua 脚本缓存。

　　关于使用 Redis 对 Lua 脚本进行求值的更多信息，请参见 `EVAL` 命令。

　　**可用版本：** >= 2.6.0

　　**复杂度：**

　　O(N) ， `N` 为缓存中脚本的数量。

　　**返回值：**

　　总是返回 `OK`

```
redis> SCRIPT FLUSH
OK
```

### SCRIPT KILL

　　**SCRIPT KILL**

　　杀死当前正在运行的 Lua 脚本，当且仅当这个脚本没有执行过任何写操作时，这个命令才生效。

　　这个命令主要用于终止运行时间过长的脚本，比如一个因为 BUG 而发生无限 loop 的脚本，诸如此类。

　　`SCRIPT KILL` 执行之后，当前正在运行的脚本会被杀死，执行这个脚本的客户端会从 `EVAL` 命令的阻塞当中退出，并收到一个错误作为返回值。

　　另一方面，假如当前正在运行的脚本已经执行过写操作，那么即使执行 `SCRIPT KILL` ，也无法将它杀死，因为这是违反 Lua 脚本的原子性执行原则的。在这种情况下，唯一可行的办法是使用 `SHUTDOWN NOSAVE` 命令，通过停止整个 Redis 进程来停止脚本的运行，并防止不完整(half-written)的信息被写入数据库中。

　　关于使用 Redis 对 Lua 脚本进行求值的更多信息，请参见 `EVAL` 命令。

　　**可用版本：** >= 2.6.0

　　**时间复杂度：**

　　O(1)

　　**返回值：**

　　执行成功返回 `OK` ，否则返回一个错误。

```
# 没有脚本在执行时

redis> SCRIPT KILL
(error) ERR No scripts in execution right now.

# 成功杀死脚本时

redis> SCRIPT KILL
OK
(1.30s)

# 尝试杀死一个已经执行过写操作的脚本，失败

redis> SCRIPT KILL
(error) ERR Sorry the script already executed write commands against the dataset. You can either wait the script termination or kill the server in an hard way using the SHUTDOWN NOSAVE command.
(1.69s)
```

　　以下是脚本被杀死之后，返回给执行脚本的客户端的错误：

```
redis> EVAL "while true do end" 0
(error) ERR Error running script (call to f_694a5fe1ddb97a4c6a1bf299d9537c7d3d0f84e7): Script killed by user with SCRIPT KILL...
(5.00s)
```

### SCRIPT LOAD

　　**SCRIPT LOAD script**

　　将脚本 `script` 添加到脚本缓存中，但并不立即执行这个脚本。

　　`EVAL` 命令也会将脚本添加到脚本缓存中，但是它会立即对输入的脚本进行求值。

　　如果给定的脚本已经在缓存里面了，那么不做动作。

　　在脚本被加入到缓存之后，通过 EVALSHA 命令，可以使用脚本的 SHA1 校验和来调用这个脚本。

　　脚本可以在缓存中保留无限长的时间，直到执行 `script_flush` 为止。

　　关于使用 Redis 对 Lua 脚本进行求值的更多信息，请参见 `EVAL` 命令。

　　**可用版本：** >= 2.6.0

　　**时间复杂度：**

　　O(N) , `N` 为脚本的长度(以字节为单位)。

　　**返回值：**

　　给定 `script` 的 SHA1 校验和

```
redis> SCRIPT LOAD "return 'hello moto'"
"232fd51614574cf0867b83d384a5e898cfd24e5a"

redis> EVALSHA 232fd51614574cf0867b83d384a5e898cfd24e5a 0
"hello moto"
```

## 自动过期

### EXPIRE key seconds

　　可用版本： >= 1.0.0

　　时间复杂度： O(1)

　　为给定 `key` 设置生存时间，当 `key` 过期时(生存时间为 0 )，它会被自动删除。

　　在 Redis 中，带有生存时间的 `key` 被称为『易失的』(volatile)。

　　生存时间可以通过使用 DEL 命令来删除整个 `key` 来移除，或者被 SET 和 GETSET 命令覆写(overwrite)，这意味着，如果一个命令只是修改(alter)一个带生存时间的 `key` 的值而不是用一个新的 `key` 值来代替(replace)它的话，那么生存时间不会被改变。

　　比如说，对一个 `key` 执行 INCR 命令，对一个列表进行 LPUSH 命令，或者对一个哈希表执行 HSET 命令，这类操作都不会修改 `key` 本身的生存时间。

　　另一方面，如果使用 RENAME 对一个 `key` 进行改名，那么改名后的 `key` 的生存时间和改名前一样。

　　RENAME 命令的另一种可能是，尝试将一个带生存时间的 `key` 改名成另一个带生存时间的 another_key ，这时旧的 another_key (以及它的生存时间)会被删除，然后旧的 `key` 会改名为 another_key ，因此，新的 another_key 的生存时间也和原本的 `key` 一样。

　　使用 PERSIST 命令可以在不删除 `key` 的情况下，移除 `key` 的生存时间，让 `key` 重新成为一个『持久的』(persistent) `key` 。

　　更新生存时间

　　可以对一个已经带有生存时间的 `key` 执行 EXPIRE 命令，新指定的生存时间会取代旧的生存时间。

　　过期时间的精确度

　　在 Redis 2.4 版本中，过期时间的延迟在 1 秒钟之内 —— 也即是，就算 `key` 已经过期，但它还是可能在过期之后一秒钟之内被访问到，而在新的 Redis 2.6 版本中，延迟被降低到 1 毫秒之内。

　　Redis 2.1.3 之前的不同之处

　　在 Redis 2.1.3 之前的版本中，修改一个带有生存时间的 `key` 会导致整个 `key` 被删除，这一行为是受当时复制(replication)层的限制而作出的，现在这一限制已经被修复。

　　**返回值**

　　设置成功返回 `1` 。 当`key`不存在或者不能为 `key`设置生存时间时(比如在低于 2.1.3 版本的 Redis 中你尝试更新 key 的生存时间)，返回 0 。

　　代码示例

```
redis> SET cache_page "www.google.com"
OK

redis> EXPIRE cache_page 30  # 设置过期时间为 30 秒
(integer) 1

redis> TTL cache_page    # 查看剩余生存时间
(integer) 23

redis> EXPIRE cache_page 30000   # 更新过期时间
(integer) 1

redis> TTL cache_page
(integer) 29996
```

　　**模式：导航会话**

　　假设你有一项 web 服务，打算根据用户最近访问的 N 个页面来进行物品推荐，并且假设用户停止阅览超过 60 秒，那么就清空阅览记录(为了减少物品推荐的计算量，并且保持推荐物品的新鲜度)。

　　这些最近访问的页面记录，我们称之为『导航会话』(Navigation session)，可以用 INCR 和 RPUSH 命令在 Redis 中实现它：每当用户阅览一个网页的时候，执行以下代码：

```
MULTI
RPUSH pagewviews.user:<userid> http://.....
EXPIRE pagewviews.user:<userid> 60
EXEC
```

　　如果用户停止阅览超过 60 秒，那么它的导航会话就会被清空，当用户重新开始阅览的时候，系统又会重新记录导航会话，继续进行物品推荐。

### EXPIREAT key timestamp

> 可用版本： >= 1.2.0
> 时间复杂度： O(1)

　　`EXPIREAT` 的作用和 `EXPIRE` 类似，都用于为 `key` 设置生存时间。

　　不同在于 `EXPIREAT` 命令接受的时间参数是 UNIX 时间戳(unix timestamp)。

　　**返回值**

　　如果生存时间设置成功，返回 `1` ； 当 `key` 不存在或没办法设置生存时间，返回 `0` 。

　　**代码示例**

```
redis> SET cache www.google.com
OK

redis> EXPIREAT cache 1355292000     # 这个 key 将在 2012.12.12 过期
(integer) 1

redis> TTL cache
(integer) 45081860
```

### TTL key

> 可用版本： >= 1.0.0
> 时间复杂度： O(1)

　　以秒为单位，返回给定 `key` 的剩余生存时间(TTL, time to live)。

　　**返回值**

　　当 `key` 不存在时，返回 `-2` 。 当 `key` 存在但没有设置剩余生存时间时，返回 `-1` 。 否则，以秒为单位，返回 `key` 的剩余生存时间。

　　Note

　　在 Redis 2.8 以前，当 `key` 不存在，或者 `key` 没有设置剩余生存时间时，命令都返回 `-1` 。

　　**代码示例**

```
# 不存在的 key

redis> FLUSHDB
OK

redis> TTL key
(integer) -2


# key 存在，但没有设置剩余生存时间

redis> SET key value
OK

redis> TTL key
(integer) -1


# 有剩余生存时间的 key

redis> EXPIRE key 10086
(integer) 1

redis> TTL key
(integer) 10084
```

### PERSIST

　　**PERSIST key**

> 可用版本： >= 2.2.0
> 时间复杂度： O(1)

　　移除给定 `key` 的生存时间，将这个 `key` 从“易失的”(带生存时间 `key` )转换成“持久的”(一个不带生存时间、永不过期的 `key` )。

　　**返回值**

　　当生存时间移除成功时，返回 `1` . 如果 `key` 不存在或 `key` 没有设置生存时间，返回 `0` 。

　　**代码示例**

```
redis> SET mykey "Hello"
OK

redis> EXPIRE mykey 10  # 为 key 设置生存时间
(integer) 1

redis> TTL mykey
(integer) 10

redis> PERSIST mykey    # 移除 key 的生存时间
(integer) 1

redis> TTL mykey
(integer) -1
```

### PEXPIRE

　　**PEXPIRE** key milliseconds

> 可用版本： >= 2.6.0
> 时间复杂度： O(1)

　　这个命令和 `EXPIRE` 命令的作用类似，但是它以毫秒为单位设置 `key` 的生存时间，而不像 `EXPIRE` 命令那样，以秒为单位。

　　**返回值**

　　设置成功，返回 `1` `key` 不存在或设置失败，返回 `0`

　　**代码示例**

```
redis> SET mykey "Hello"
OK

redis> PEXPIRE mykey 1500
(integer) 1

redis> TTL mykey    # TTL 的返回值以秒为单位
(integer) 2

redis> PTTL mykey   # PTTL 可以给出准确的毫秒数
(integer) 1499
```

### PEXPIREAT

　　**PEXPIREAT key milliseconds-timestamp**

> 可用版本： >= 2.6.0
> 时间复杂度： O(1)

　　这个命令和 `expireat` 命令类似，但它以毫秒为单位设置 `key` 的过期 unix 时间戳，而不是像 `expireat` 那样，以秒为单位。

　　**返回值**

　　如果生存时间设置成功，返回 `1` 。 当 `key` 不存在或没办法设置生存时间时，返回 `0` 。(查看 [EXPIRE key seconds](http://redisdoc.com/expire/expire.html) 命令获取更多信息)

　　**代码示例**

```
redis> SET mykey "Hello"
OK

redis> PEXPIREAT mykey 1555555555005
(integer) 1

redis> TTL mykey           # TTL 返回秒
(integer) 223157079

redis> PTTL mykey          # PTTL 返回毫秒
(integer) 223157079318
```

### PTTL

　　**PTTL key**

> 可用版本： >= 2.6.0
> 复杂度： O(1)

　　这个命令类似于 `TTL` 命令，但它以毫秒为单位返回 `key` 的剩余生存时间，而不是像 `TTL` 命令那样，以秒为单位。

　　**返回值**

- 当 `key` 不存在时，返回 `-2` 。
- 当 `key` 存在但没有设置剩余生存时间时，返回 `-1` 。
- 否则，以毫秒为单位，返回 `key` 的剩余生存时间。

　　Note

　　在 Redis 2.8 以前，当 `key` 不存在，或者 `key` 没有设置剩余生存时间时，命令都返回 `-1` 。

　　**代码示例**

```
# 不存在的 key

redis> FLUSHDB
OK

redis> PTTL key
(integer) -2


# key 存在，但没有设置剩余生存时间

redis> SET key value
OK

redis> PTTL key
(integer) -1


# 有剩余生存时间的 key

redis> PEXPIRE key 10086
(integer) 1

redis> PTTL key
(integer) 6179
```

## 地理位置

### GEOADD

　　**GEOADD key longitude latitude member [longitude latitude member …]**

> 可用版本： >= 3.2.0
> 时间复杂度： 每添加一个元素的复杂度为 O(log(N)) ， 其中 N 为键里面包含的位置元素数量。

　　将给定的空间元素（纬度、经度、名字）添加到指定的键里面。 这些数据会以有序集合的形式被储存在键里面， 从而使得像 `GEORADIUS` 和 `GEORADIUSBYMEMBER` 这样的命令可以在之后通过位置查询取得这些元素。

　　`GEOADD` 命令以标准的 `x,y` 格式接受参数， 所以用户必须先输入经度， 然后再输入纬度。 `GEOADD` 能够记录的坐标是有限的： 非常接近两极的区域是无法被索引的。 精确的坐标限制由 EPSG:900913 / EPSG:3785 / OSGEO:41001 等坐标系统定义， 具体如下：

- 有效的经度介于 -180 度至 180 度之间。
- 有效的纬度介于 -85.05112878 度至 85.05112878 度之间。

　　当用户尝试输入一个超出范围的经度或者纬度时， `GEOADD` 命令将返回一个错误。

　　**返回值**

　　新添加到键里面的空间元素数量， 不包括那些已经存在但是被更新的元素。

　　**代码示例**

```
redis> GEOADD Sicily 13.361389 38.115556 "Palermo" 15.087269 37.502669 "Catania"
(integer) 2

redis> GEODIST Sicily Palermo Catania
"166274.15156960039"

redis> GEORADIUS Sicily 15 37 100 km
1) "Catania"

redis> GEORADIUS Sicily 15 37 200 km
1) "Palermo"
2) "Catania"
```

### GEOPOS

　　**GEOPOS key member [member …]**

> 可用版本： >= 3.2.0
> 时间复杂度： 获取每个位置元素的复杂度为 O(log(N)) ， 其中 N 为键里面包含的位置元素数量。

　　从键里面返回所有给定位置元素的位置（经度和纬度）。

　　因为 `GEOPOS` 命令接受可变数量的位置元素作为输入， 所以即使用户只给定了一个位置元素， 命令也会返回数组回复。

　　**返回值**

　　`GEOPOS` 命令返回一个数组， 数组中的每个项都由两个元素组成： 第一个元素为给定位置元素的经度， 而第二个元素则为给定位置元素的纬度。 当给定的位置元素不存在时， 对应的数组项为空值。

　　**代码示例**

```
redis> GEOADD Sicily 13.361389 38.115556 "Palermo" 15.087269 37.502669 "Catania"
(integer) 2

redis> GEOPOS Sicily Palermo Catania NonExisting
1) 1) "13.361389338970184"
   1) "38.115556395496299"
2) 1) "15.087267458438873"
   1) "37.50266842333162"
3) (nil)
```

### GEODIST

　　**GEODIST key member1 member2 [unit]**

> 可用版本： >= 3.2.0
> 复杂度： O(log(N))

　　返回两个给定位置之间的距离。

　　如果两个位置之间的其中一个不存在， 那么命令返回空值。

　　指定单位的参数 `unit` 必须是以下单位的其中一个：

- `m` 表示单位为米。
- `km` 表示单位为千米。
- `mi` 表示单位为英里。
- `ft` 表示单位为英尺。

　　如果用户没有显式地指定单位参数， 那么 `GEODIST` 默认使用米作为单位。

　　`GEODIST` 命令在计算距离时会假设地球为完美的球形， 在极限情况下， 这一假设最大会造成 0.5% 的误差。

　　**返回值**

　　计算出的距离会以双精度浮点数的形式被返回。 如果给定的位置元素不存在， 那么命令返回空值。

　　**代码示例**

```
redis> GEOADD Sicily 13.361389 38.115556 "Palermo" 15.087269 37.502669 "Catania"
(integer) 2

redis> GEODIST Sicily Palermo Catania
"166274.15156960039"

redis> GEODIST Sicily Palermo Catania km
"166.27415156960038"

redis> GEODIST Sicily Palermo Catania mi
"103.31822459492736"

redis> GEODIST Sicily Foo Bar
(nil)
```

### GEORADIUS

　　**GEORADIUS key longitude latitude radius m|km|ft|mi [WITHCOORD] [WITHDIST] [WITHHASH] [ASC|DESC] [COUNT count]**

> 可用版本： >= 3.2.0
> 时间复杂度： O(N+log(M))， 其中 N 为指定半径范围内的位置元素数量， 而 M 则是被返回位置元素的数量。

　　以给定的经纬度为中心， 返回键包含的位置元素当中， 与中心的距离不超过给定最大距离的所有位置元素。

　　范围可以使用以下其中一个单位：

- `m` 表示单位为米。
- `km` 表示单位为千米。
- `mi` 表示单位为英里。
- `ft` 表示单位为英尺。

　　在给定以下可选项时， 命令会返回额外的信息：

- `WITHDIST` ： 在返回位置元素的同时， 将位置元素与中心之间的距离也一并返回。 距离的单位和用户给定的范围单位保持一致。
- `WITHCOORD` ： 将位置元素的经度和维度也一并返回。
- `WITHHASH` ： 以 52 位有符号整数的形式， 返回位置元素经过原始 geohash 编码的有序集合分值。 这个选项主要用于底层应用或者调试， 实际中的作用并不大。

　　命令默认返回未排序的位置元素。 通过以下两个参数， 用户可以指定被返回位置元素的排序方式：

- `ASC` ： 根据中心的位置， 按照从近到远的方式返回位置元素。
- `DESC` ： 根据中心的位置， 按照从远到近的方式返回位置元素。

　　在默认情况下， `GEORADIUS` 命令会返回所有匹配的位置元素。 虽然用户可以使用 `COUNT` 选项去获取前 N 个匹配元素， 但是因为命令在内部可能会需要对所有被匹配的元素进行处理， 所以在对一个非常大的区域进行搜索时， 即使只使用 `COUNT` 选项去获取少量元素， 命令的执行速度也可能会非常慢。 但是从另一方面来说， 使用 `COUNT` 选项去减少需要返回的元素数量， 对于减少带宽来说仍然是非常有用的。

　　**返回值**

　　`GEORADIUS` 命令返回一个数组， 具体来说：

- 在没有给定任何 `WITH` 选项的情况下， 命令只会返回一个像 `["New York","Milan","Paris"]` 这样的线性（linear）列表。
- 在指定了 `WITHCOORD` 、 `WITHDIST` 、 `WITHHASH` 等选项的情况下， 命令返回一个二层嵌套数组， 内层的每个子数组就表示一个元素。

　　在返回嵌套数组时， 子数组的第一个元素总是位置元素的名字。 至于额外的信息， 则会作为子数组的后续元素， 按照以下顺序被返回：

1. 以浮点数格式返回的中心与位置元素之间的距离， 单位与用户指定范围时的单位一致。
2. geohash 整数。
3. 由两个元素组成的坐标，分别为经度和纬度。

　　举个例子， `GEORADIUS Sicily 15 37 200 km withcoord withdist` 这样的命令返回的每个子数组都是类似以下格式的：

```
["Palermo","190.4424",["13.361389338970184","38.115556395496299"]]
```

　　**代码示例**

```
redis> GEOADD Sicily 13.361389 38.115556 "Palermo" 15.087269 37.502669 "Catania"
(integer) 2

redis> GEORADIUS Sicily 15 37 200 km WITHDIST
1) 1) "Palermo"
   2) "190.4424"
2) 1) "Catania"
   2) "56.4413"

redis> GEORADIUS Sicily 15 37 200 km WITHCOORD
1) 1) "Palermo"
   2) 1) "13.361389338970184"
      2) "38.115556395496299"
2) 1) "Catania"
   2) 1) "15.087267458438873"
      2) "37.50266842333162"

redis> GEORADIUS Sicily 15 37 200 km WITHDIST WITHCOORD
1) 1) "Palermo"
   2) "190.4424"
   3) 1) "13.361389338970184"
      2) "38.115556395496299"
2) 1) "Catania"
   2) "56.4413"
   3) 1) "15.087267458438873"
      2) "37.50266842333162"
```

### GEORADIUSBYMEMBER

　　**GEORADIUSBYMEMBER key member radius m|km|ft|mi [WITHCOORD] [WITHDIST] [WITHHASH] [ASC|DESC] [COUNT count]**

> 可用版本： >= 3.2.0
> 时间复杂度： O(log(N)+M)， 其中 N 为指定范围之内的元素数量， 而 M 则是被返回的元素数量。

　　这个命令和 `GEORADIUS` 命令一样， 都可以找出位于指定范围内的元素， 但是 `GEORADIUSBYMEMBER` 的中心点是由给定的位置元素决定的， 而不是像 `GEORADIUS` 那样， 使用输入的经度和纬度来决定中心点。

　　关于 `GEORADIUSBYMEMBER` 命令的更多信息， 请参考 `GEORADIUS` 命令的文档。

　　**返回值**

　　一个数组， 数组中的每个项表示一个范围之内的位置元素。

　　**代码示例**

```
redis> GEOADD Sicily 13.583333 37.316667 "Agrigento"
(integer) 1

redis> GEOADD Sicily 13.361389 38.115556 "Palermo" 15.087269 37.502669 "Catania"
(integer) 2

redis> GEORADIUSBYMEMBER Sicily Agrigento 100 km
1) "Agrigento"
2) "Palermo"
```

### GEOHASH

　　**GEOHASH key member [member …]**

> 可用版本： >= 3.2.0
> 时间复杂度： 寻找每个位置元素的复杂度为 O(log(N)) ， 其中 N 为给定键包含的位置元素数量。

　　返回一个或多个位置元素的 [Geohash](https://en.wikipedia.org/wiki/Geohash) 表示。

　　**返回值**

　　一个数组， 数组的每个项都是一个 geohash 。 命令返回的 geohash 的位置与用户给定的位置元素的位置一一对应。

　　**代码示例**

```
redis> GEOADD Sicily 13.361389 38.115556 "Palermo" 15.087269 37.502669 "Catania"
(integer) 2

redis> GEOHASH Sicily Palermo Catania
1) "sqc8b49rny0"
2) "sqdtr74hyu0"
```

## 位图

### SETBIT

　　**SETBIT key offset value**

　　对 `key` 所储存的字符串值，设置或清除指定偏移量上的位(bit)。

　　位的设置或清除取决于 `value` 参数，可以是 `0` 也可以是 `1` 。

　　当 `key` 不存在时，自动生成一个新的字符串值。

　　字符串会进行伸展(grown)以确保它可以将 `value` 保存在指定的偏移量上。当字符串值进行伸展时，空白位置以 `0` 填充。

　　`offset` 参数必须大于或等于 `0` ，小于 2^32 (bit 映射被限制在 512 MB 之内)。

> warning:: 对使用大的 `offset` 的 `SETBIT` 操作来说，内存分配可能造成 Redis 服务器被阻塞。具体参考 `SETRANGE` 命令，warning(警告)部分。

　　**可用版本：** >= 2.2.0

　　**时间复杂度:**

　　O(1)

　　**返回值：**

　　指定偏移量原来储存的位。

```
redis> SETBIT bit 10086 1
(integer) 0

redis> GETBIT bit 10086
(integer) 1

redis> GETBIT bit 100   # bit 默认被初始化为 0
(integer) 0
```

### GETBIT

　　**GETBIT key offset**

　　对 `key` 所储存的字符串值，获取指定偏移量上的位(bit)。

　　当 `offset` 比字符串值的长度大，或者 `key` 不存在时，返回 `0` 。

　　**可用版本：** >= 2.2.0

　　**时间复杂度：**

　　O(1)

　　**返回值：**

　　字符串值指定偏移量上的位(bit)。

```
# 对不存在的 key 或者不存在的 offset 进行 GETBIT， 返回 0

redis> EXISTS bit
(integer) 0

redis> GETBIT bit 10086
(integer) 0


# 对已存在的 offset 进行 GETBIT

redis> SETBIT bit 10086 1
(integer) 0

redis> GETBIT bit 10086
(integer) 1
```

### BITCOUNT

　　**BITCOUNT key [start][end]**

　　计算给定字符串中，被设置为 `1` 的比特位的数量。

　　一般情况下，给定的整个字符串都会被进行计数，通过指定额外的 `start` 或 `end` 参数，可以让计数只在特定的位上进行。

　　`start` 和 `end` 参数的设置和 `getrange` 命令类似，都可以使用负数值：

　　比如 `-1` 表示最后一个字节， `-2` 表示倒数第二个字节，以此类推。

　　不存在的 `key` 被当成是空字符串来处理，因此对一个不存在的 `key` 进行 `BITCOUNT` 操作，结果为 `0` 。

　　**可用版本：** >= 2.6.0

　　**时间复杂度：**

　　O(N)

　　**返回值：**

　　被设置为 `1` 的位的数量。

```
redis> BITCOUNT bits
(integer) 0

redis> SETBIT bits 0 1          # 0001
(integer) 0

redis> BITCOUNT bits
(integer) 1

redis> SETBIT bits 3 1          # 1001
(integer) 0

redis> BITCOUNT bits
(integer) 2
```

　　**模式：使用 bitmap 实现用户上线次数统计**

　　Bitmap 对于一些特定类型的计算非常有效。

　　假设现在我们希望记录自己网站上的用户的上线频率，比如说，计算用户 A 上线了多少天，用户 B 上线了多少天，诸如此类，以此作为数据，从而决定让哪些用户参加 beta 测试等活动 —— 这个模式可以使用 `SETBIT` 和 `BITCOUNT` 来实现。

　　比如说，每当用户在某一天上线的时候，我们就使用 `SETBIT` ，以用户名作为 `key` ，将那天所代表的网站的上线日作为 `offset` 参数，并将这个 `offset` 上的为设置为 `1` 。

　　举个例子，如果今天是网站上线的第 100 天，而用户 peter 在今天阅览过网站，那么执行命令 `SETBIT peter 100 1` ；如果明天 peter 也继续阅览网站，那么执行命令 `SETBIT peter 101 1` ，以此类推。

　　当要计算 peter 总共以来的上线次数时，就使用 `BITCOUNT` 命令：执行 `BITCOUNT peter` ，得出的结果就是 peter 上线的总天数。

　　更详细的实现可以参考博文(墙外) `Fast, easy, realtime metrics using Redis bitmaps <http://blog.getspool.com/2011/11/29/fast-easy-realtime-metrics-using-redis-bitmaps/>` 。

　　**性能**

　　前面的上线次数统计例子，即使运行 10 年，占用的空间也只是每个用户 10*365 比特位(bit)，也即是每个用户 456 字节。对于这种大小的数据来说， `BITCOUNT` 的处理速度就像 `GET` 和 `INCR` 这种 O(1) 复杂度的操作一样快。

　　如果你的 bitmap 数据非常大，那么可以考虑使用以下两种方法：

1. 将一个大的 bitmap 分散到不同的 key 中，作为小的 bitmap 来处理。使用 Lua 脚本可以很方便地完成这一工作。
2. 使用 `BITCOUNT` 的 `start` 和 `end` 参数，每次只对所需的部分位进行计算，将位的累积工作(accumulating)放到客户端进行，并且对结果进行缓存 (caching)。

### BITOP

　　**BITOP operation destkey key [key ...]**

　　对一个或多个保存二进制位的字符串 `key` 进行位元操作，并将结果保存到 `destkey` 上。

　　`operation` 可以是 `AND` 、 `OR` 、 `NOT` 、 `XOR` 这四种操作中的任意一种：

- `BITOP AND destkey key [key ...]` ，对一个或多个 `key` 求逻辑并，并将结果保存到 `destkey` 。
- `BITOP OR destkey key [key ...]` ，对一个或多个 `key` 求逻辑或，并将结果保存到 `destkey` 。
- `BITOP XOR destkey key [key ...]` ，对一个或多个 `key` 求逻辑异或，并将结果保存到 `destkey` 。
- `BITOP NOT destkey key` ，对给定 `key` 求逻辑非，并将结果保存到 `destkey` 。

　　除了 `NOT` 操作之外，其他操作都可以接受一个或多个 `key` 作为输入。

　　**处理不同长度的字符串**

　　当 `BITOP` 处理不同长度的字符串时，较短的那个字符串所缺少的部分会被看作 `0` 。

　　空的 `key` 也被看作是包含 `0` 的字符串序列。

　　**可用版本：** >= 2.6.0

　　**时间复杂度：**

　　O(N)

　　**返回值：**

　　保存到 `destkey` 的字符串的长度，和输入 `key` 中最长的字符串长度相等。

　　`BITOP` 的复杂度为 O(N) ，当处理大型矩阵(matrix)或者进行大数据量的统计时，最好将任务指派到附属节点(slave)进行，避免阻塞主节点。

```
redis> SETBIT bits-1 0 1        # bits-1 = 1001
(integer) 0

redis> SETBIT bits-1 3 1
(integer) 0

redis> SETBIT bits-2 0 1        # bits-2 = 1011
(integer) 0

redis> SETBIT bits-2 1 1
(integer) 0

redis> SETBIT bits-2 3 1
(integer) 0

redis> BITOP AND and-result bits-1 bits-2
(integer) 1

redis> GETBIT and-result 0      # and-result = 1001
(integer) 1

redis> GETBIT and-result 1
(integer) 0

redis> GETBIT and-result 2
(integer) 0

redis> GETBIT and-result 3
(integer) 1
```

### BITPOS

　　**BITPOS key bit [start] [end]**

> 可用版本： >= 2.8.7
> 时间复杂度： O(N)，其中 N 为位图包含的二进制位数量

　　返回位图中第一个值为 `bit` 的二进制位的位置。

　　在默认情况下， 命令将检测整个位图， 但用户也可以通过可选的 `start` 参数和 `end` 参数指定要检测的范围。

　　**返回值**

　　整数回复。

　　**代码示例**

```
127.0.0.1:6379> SETBIT bits 3 1    # 1000
(integer) 0

127.0.0.1:6379> BITPOS bits 0
(integer) 0

127.0.0.1:6379> BITPOS bits 1
(integer) 3
```

### BITFIELD

　　**BITFIELD key [GET type offset] [SET type offset value] [INCRBY type offset increment] [OVERFLOW WRAP|SAT|FAIL]**

> 可用版本： >= 3.2.0
> 时间复杂度： 每个子命令的复杂度为 O(1) 。

　　`BITFIELD` 命令可以将一个 Redis 字符串看作是一个由二进制位组成的数组， 并对这个数组中储存的长度不同的整数进行访问 （被储存的整数无需进行对齐）。 换句话说， 通过这个命令， 用户可以执行诸如 “对偏移量 1234 上的 5 位长有符号整数进行设置”、 “获取偏移量 4567 上的 31 位长无符号整数”等操作。 此外， `BITFIELD` 命令还可以对指定的整数执行加法操作和减法操作， 并且这些操作可以通过设置妥善地处理计算时出现的溢出情况。

　　`BITFIELD` 命令可以在一次调用中同时对多个位范围进行操作： 它接受一系列待执行的操作作为参数， 并返回一个数组作为回复， 数组中的每个元素就是对应操作的执行结果。

　　比如以下命令就展示了如何对位于偏移量 100 的 8 位长有符号整数执行加法操作， 并获取位于偏移量 0 上的 4 位长无符号整数：

```
> BITFIELD mykey INCRBY i8 100 1 GET u4 0
1) (integer) 1
2) (integer) 0
```

　　注意：

- 使用 `GET` 子命令对超出字符串当前范围的二进制位进行访问（包括键不存在的情况）， 超出部分的二进制位的值将被当做是 0 。
- 使用 `SET` 子命令或者 `INCRBY` 子命令对超出字符串当前范围的二进制位进行访问将导致字符串被扩大， 被扩大的部分会使用值为 0 的二进制位进行填充。 在对字符串进行扩展时， 命令会根据字符串目前已有的最远端二进制位， 计算出执行操作所需的最小长度。

　　**支持的子命令以及数字类型**

　　以下是 `BITFIELD` 命令支持的子命令：

- `GET` —— 返回指定的二进制位范围。
- `SET` —— 对指定的二进制位范围进行设置，并返回它的旧值。
- `INCRBY` —— 对指定的二进制位范围执行加法操作，并返回它的旧值。用户可以通过向 `increment` 参数传入负值来实现相应的减法操作。

　　除了以上三个子命令之外， 还有一个子命令， 它可以改变之后执行的 `INCRBY` 子命令在发生溢出情况时的行为：

- `OVERFLOW [WRAP|SAT|FAIL]`

　　当被设置的二进制位范围值为整数时， 用户可以在类型参数的前面添加 `i` 来表示有符号整数， 或者使用 `u` 来表示无符号整数。 比如说， 我们可以使用 `u8` 来表示 8 位长的无符号整数， 也可以使用 `i16` 来表示 16 位长的有符号整数。

　　`BITFIELD` 命令最大支持 64 位长的有符号整数以及 63 位长的无符号整数， 其中无符号整数的 63 位长度限制是由于 Redis 协议目前还无法返回 64 位长的无符号整数而导致的。

　　**二进制位和位置偏移量**

　　在二进制位范围命令中， 用户有两种方法来设置偏移量：

- 如果用户给定的是一个没有任何前缀的数字， 那么这个数字指示的就是字符串以零为开始（zero-base）的偏移量。
- 另一方面， 如果用户给定的是一个带有 `#` 前缀的偏移量， 那么命令将使用这个偏移量与被设置的数字类型的位长度相乘， 从而计算出真正的偏移量。

　　比如说， 对于以下这个命令来说：

```
BITFIELD mystring SET i8 #0 100 i8 #1 200
```

　　命令会把 `mystring` 键里面， 第一个 `i8` 长度的二进制位的值设置为 `100` ， 并把第二个 `i8` 长度的二进制位的值设置为 `200` 。 当我们把一个字符串键当成数组来使用， 并且数组中储存的都是同等长度的整数时， 使用 `#` 前缀可以让我们免去手动计算被设置二进制位所在位置的麻烦。

　　**溢出控制**

　　用户可以通过 `OVERFLOW` 命令以及以下展示的三个参数， 指定 `BITFIELD` 命令在执行自增或者自减操作时， 碰上向上溢出（overflow）或者向下溢出（underflow）情况时的行为：

- `WRAP` ： 使用回绕（wrap around）方法处理有符号整数和无符号整数的溢出情况。 对于无符号整数来说， 回绕就像使用数值本身与能够被储存的最大无符号整数执行取模计算， 这也是 C 语言的标准行为。 对于有符号整数来说， 上溢将导致数字重新从最小的负数开始计算， 而下溢将导致数字重新从最大的正数开始计算。 比如说， 如果我们对一个值为 `127` 的 `i8` 整数执行加一操作， 那么将得到结果 `-128` 。
- `SAT` ： 使用饱和计算（saturation arithmetic）方法处理溢出， 也即是说， 下溢计算的结果为最小的整数值， 而上溢计算的结果为最大的整数值。 举个例子， 如果我们对一个值为 `120` 的 `i8` 整数执行加 `10` 计算， 那么命令的结果将为 `i8` 类型所能储存的最大整数值 `127` 。 与此相反， 如果一个针对 `i8` 值的计算造成了下溢， 那么这个 `i8` 值将被设置为 `-127` 。
- `FAIL` ： 在这一模式下， 命令将拒绝执行那些会导致上溢或者下溢情况出现的计算， 并向用户返回空值表示计算未被执行。

　　需要注意的是， `OVERFLOW` 子命令只会对紧随着它之后被执行的 `INCRBY` 命令产生效果， 这一效果将一直持续到与它一同被执行的下一个 `OVERFLOW` 命令为止。 在默认情况下， `INCRBY` 命令使用 `WRAP` 方式来处理溢出计算。

　　以下是一个使用 `OVERFLOW` 子命令来控制溢出行为的例子：

```
> BITFIELD mykey incrby u2 100 1 OVERFLOW SAT incrby u2 102 1
1) (integer) 1
2) (integer) 1

> BITFIELD mykey incrby u2 100 1 OVERFLOW SAT incrby u2 102 1
1) (integer) 2
2) (integer) 2

> BITFIELD mykey incrby u2 100 1 OVERFLOW SAT incrby u2 102 1
1) (integer) 3
2) (integer) 3

> BITFIELD mykey incrby u2 100 1 OVERFLOW SAT incrby u2 102 1
1) (integer) 0  -- 使用默认的 WRAP 方式处理溢出
2) (integer) 3  -- 使用 SAT 方式处理溢出
```

　　而以下则是一个因为 `OVERFLOW FAIL` 行为而导致子命令返回空值的例子：

```
> BITFIELD mykey OVERFLOW FAIL incrby u2 102 1
1) (nil)
```

　　**作用**

　　`BITFIELD` 命令的作用在于它能够将很多小的整数储存到一个长度较大的位图中， 又或者将一个非常庞大的键分割为多个较小的键来进行储存， 从而非常高效地使用内存， 使得 Redis 能够得到更多不同的应用 —— 特别是在实时分析领域： `BITFIELD` 能够以指定的方式对计算溢出进行控制的能力， 使得它可以被应用于这一领域。

　　**性能注意事项**

　　`BITFIELD` 在一般情况下都是一个快速的命令， 需要注意的是， 访问一个长度较短的字符串的远端二进制位将引发一次内存分配操作， 这一操作花费的时间可能会比命令访问已有的字符串花费的时间要长。

　　**二进制位的排列**

　　`BITFIELD` 把位图第一个字节偏移量 0 上的二进制位看作是 most significant 位， 以此类推。 举个例子， 如果我们对一个已经预先被全部设置为 0 的位图进行设置， 将它在偏移量 7 的值设置为 5 位无符号整数值 23 （二进制位为 `10111` ）， 那么命令将生产出以下这个位图表示：

```
+--------+--------+
|00000001|01110000|
+--------+--------+
```

　　当偏移量和整数长度与字节边界进行对齐时， `BITFIELD` 表示二进制位的方式跟大端表示法（big endian）一致， 但是在没有对齐的情况下， 理解这些二进制位是如何进行排列也是非常重要的。

　　**返回值**

　　`BITFIELD` 命令的返回值是一个数组， 数组中的每个元素对应一个被执行的子命令。 需要注意的是， `OVERFLOW` 子命令本身并不产生任何回复。

## 内部命令

### MIGRATE

　　**MIGRATE host port key destination-db timeout [COPY] [REPLACE]**

> 可用版本： >= 2.6.0
> 时间复杂度： 这个命令在源实例上实际执行 `DUMP` 命令和 `DEL` 命令，在目标实例执行 `RESTORE` 命令，查看以上命令的文档可以看到详细的复杂度说明。 `key` 数据在两个实例之间传输的复杂度为 O(N) 。

　　将 `key` 原子性地从当前实例传送到目标实例的指定数据库上，一旦传送成功， `key` 保证会出现在目标实例上，而当前实例上的 `key` 会被删除。

　　这个命令是一个原子操作，它在执行的时候会阻塞进行迁移的两个实例，直到以下任意结果发生：迁移成功，迁移失败，等待超时。

　　命令的内部实现是这样的：它在当前实例对给定 `key` 执行 `DUMP` 命令 ，将它序列化，然后传送到目标实例，目标实例再使用 `RESTORE` 对数据进行反序列化，并将反序列化所得的数据添加到数据库中；当前实例就像目标实例的客户端那样，只要看到 `RESTORE` 命令返回 `OK` ，它就会调用 `DEL` 删除自己数据库上的 `key` 。

　　`timeout` 参数以毫秒为格式，指定当前实例和目标实例进行沟通的**最大间隔时间**。这说明操作并不一定要在 `timeout` 毫秒内完成，只是说数据传送的时间不能超过这个 `timeout` 数。

　　`MIGRATE` 命令需要在给定的时间规定内完成 IO 操作。如果在传送数据时发生 IO 错误，或者达到了超时时间，那么命令会停止执行，并返回一个特殊的错误： `IOERR` 。

　　当 `IOERR` 出现时，有以下两种可能：

- `key` 可能存在于两个实例
- `key` 可能只存在于当前实例

　　唯一不可能发生的情况就是丢失 `key` ，因此，如果一个客户端执行 `MIGRATE` 命令，并且不幸遇上 `IOERR` 错误，那么这个客户端唯一要做的就是检查自己数据库上的 `key` 是否已经被正确地删除。

　　如果有其他错误发生，那么 `MIGRATE` 保证 `key` 只会出现在当前实例中。（当然，目标实例的给定数据库上可能有和 `key` 同名的键，不过这和 `MIGRATE` 命令没有关系）。

　　**可选项**

- `COPY` ：不移除源实例上的 `key` 。
- `REPLACE` ：替换目标实例上已存在的 `key` 。

　　**返回值**

　　迁移成功时返回 `OK` ，否则返回相应的错误。

　　**代码示例**

　　先启动两个 Redis 实例，一个使用默认的 6379 端口，一个使用 7777 端口。

```
$ ./redis-server &
[1] 3557

...

$ ./redis-server --port 7777 &
[2] 3560

...
```

　　然后用客户端连上 6379 端口的实例，设置一个键，然后将它迁移到 7777 端口的实例上：

```
$ ./redis-cli

redis 127.0.0.1:6379> flushdb
OK

redis 127.0.0.1:6379> SET greeting "Hello from 6379 instance"
OK

redis 127.0.0.1:6379> MIGRATE 127.0.0.1 7777 greeting 0 1000
OK

redis 127.0.0.1:6379> EXISTS greeting                           # 迁移成功后 key 被删除
(integer) 0
```

　　使用另一个客户端，查看 7777 端口上的实例：

```
$ ./redis-cli -p 7777

redis 127.0.0.1:7777> GET greeting
"Hello from 6379 instance"
```

### DUMP

　　**DUMP key**

> 可用版本： >= 2.6.0
> 时间复杂度：查找给定键的复杂度为 O(1) ，对键进行序列化的复杂度为 O(N*M) ，其中 N 是构成 `key` 的 Redis 对象的数量，而 M 则是这些对象的平均大小。如果序列化的对象是比较小的字符串，那么复杂度为 O(1) 。

　　序列化给定 `key` ，并返回被序列化的值，使用 `RESTORE` 命令可以将这个值反序列化为 Redis 键。

　　序列化生成的值有以下几个特点：

- 它带有 64 位的校验和，用于检测错误， `RESTORE` 在进行反序列化之前会先检查校验和。
- 值的编码格式和 RDB 文件保持一致。
- RDB 版本会被编码在序列化值当中，如果因为 Redis 的版本不同造成 RDB 格式不兼容，那么 Redis 会拒绝对这个值进行反序列化操作。

　　序列化的值不包括任何生存时间信息。

　　**返回值**

　　如果 `key` 不存在，那么返回 `nil` 。 否则，返回序列化之后的值。

　　**代码示例**

```
redis> SET greeting "hello, dumping world!"
OK

redis> DUMP greeting
"\x00\x15hello, dumping world!\x06\x00E\xa0Z\x82\xd8r\xc1\xde"

redis> DUMP not-exists-key
(nil)
```

### RESTORE

　　**RESTORE key ttl serialized-value [REPLACE]**

> 可用版本： >= 2.6.0
> 时间复杂度： 查找给定键的复杂度为 O(1) ，对键进行反序列化的复杂度为 O(N_M) ，其中 N 是构成 `key` 的 Redis 对象的数量，而 M 则是这些对象的平均大小。 有序集合(sorted set)的反序列化复杂度为 O(N_M*log(N)) ，因为有序集合每次插入的复杂度为 O(log(N)) 。 如果反序列化的对象是比较小的字符串，那么复杂度为 O(1) 。

　　反序列化给定的序列化值，并将它和给定的 `key` 关联。

　　参数 `ttl` 以毫秒为单位为 `key` 设置生存时间；如果 `ttl` 为 `0` ，那么不设置生存时间。

　　`RESTORE` 在执行反序列化之前会先对序列化值的 RDB 版本和数据校验和进行检查，如果 RDB 版本不相同或者数据不完整的话，那么 `RESTORE` 会拒绝进行反序列化，并返回一个错误。

　　如果键 `key` 已经存在， 并且给定了 `REPLACE` 选项， 那么使用反序列化得出的值来代替键 `key` 原有的值； 相反地， 如果键 `key` 已经存在， 但是没有给定 `REPLACE` 选项， 那么命令返回一个错误。

　　更多信息可以参考 `DUMP` 命令。

　　**返回值**

　　如果反序列化成功那么返回 `OK` ，否则返回一个错误。

　　**代码示例**

```
# 创建一个键，作为 DUMP 命令的输入

redis> SET greeting "hello, dumping world!"
OK

redis> DUMP greeting
"\x00\x15hello, dumping world!\x06\x00E\xa0Z\x82\xd8r\xc1\xde"

# 将序列化数据 RESTORE 到另一个键上面

redis> RESTORE greeting-again 0 "\x00\x15hello, dumping world!\x06\x00E\xa0Z\x82\xd8r\xc1\xde"
OK

redis> GET greeting-again
"hello, dumping world!"

# 在没有给定 REPLACE 选项的情况下，再次尝试反序列化到同一个键，失败

redis> RESTORE greeting-again 0 "\x00\x15hello, dumping world!\x06\x00E\xa0Z\x82\xd8r\xc1\xde"
(error) ERR Target key name is busy.

# 给定 REPLACE 选项，对同一个键进行反序列化成功

redis> RESTORE greeting-again 0 "\x00\x15hello, dumping world!\x06\x00E\xa0Z\x82\xd8r\xc1\xde" REPLACE
OK

# 尝试使用无效的值进行反序列化，出错

redis> RESTORE fake-message 0 "hello moto moto blah blah"
(error) ERR DUMP payload version or checksum are wrong
```

### SYNC

　　**SYNC**

> 可用版本： >= 1.0.0
> 时间复杂度： O(N)

　　用于复制功能(replication)的内部命令。

　　更多信息请参考 [Redis 官网的 Replication 章节](http://redis.io/topics/replication) 。

　　**返回值**

　　序列化数据。

　　**代码示例**

```
redis> SYNC
"REDIS0002\xfe\x00\x00\auser_id\xc0\x03\x00\anumbers\xc2\xf3\xe0\x01\x00\x00\tdb_number\xc0\x00\x00\x04name\x06huangz\x00\anew_key\nhello_moto\x00\bgreeting\nhello moto\x00\x05my_pc\bthinkpad\x00\x04lock\xc0\x01\x00\nlock_times\xc0\x04\xfe\x01\t\x04info\x19\x02\x04name\b\x00zhangyue\x03age\x02\x0022\xff\t\aooredis,\x03\x04name\a\x00ooredis\aversion\x03\x001.0\x06author\x06\x00huangz\xff\x00\tdb_number\xc0\x01\x00\x05greet\x0bhello world\x02\nmy_friends\x02\x05marry\x04jack\x00\x04name\x05value\xfe\x02\x0c\x01s\x12\x12\x00\x00\x00\r\x00\x00\x00\x02\x00\x00\x01a\x03\xc0f'\xff\xff"
(1.90s)
```

### PSYNC

　　**PSYNC master_run_id offset**

> 可用版本： >= 2.8.0
> 时间复杂度： 不明确

　　用于复制功能(replication)的内部命令。

　　更多信息请参考 [复制（Replication）](http://redisdoc.com/topic/replication.html#replication-topic) 文档。

　　**返回值**

　　序列化数据。

　　**代码示例**

```
127.0.0.1:6379> PSYNC ? -1
"REDIS0006\xfe\x00\x00\x02kk\x02vv\x00\x03msg\x05hello\xff\xc3\x96P\x12h\bK\xef"
```
