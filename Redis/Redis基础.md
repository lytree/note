---
title: Redis基础
date: 2022-10-30T15:05:03Z
lastmod: 2022-10-30T15:29:10Z
---

# Redis基础

---

## String（字符串）

　　底层数据结构：动态字符串（SDS）类似动态数字ArrayList

### APPEND

　　**APPEND key value**
如果 `key` 已经存在并且是一个字符串， `APPEND` 命令将 `value` 追加到 `key` 原来的值的末尾。

　　如果 `key` 不存在， `APPEND` 就简单地将给定 `key` 设为 `value` ，就像执行 `SET key value` 一样。

　　**可用版本：** >= 2.0.0

　　**时间复杂度：**
平摊 O(1)

　　**返回值：**
追加 `value` 之后， `key` 中字符串的长度。

```
# 对不存在的 key 执行 APPEND

redis> EXISTS myphone               # 确保 myphone 不存在
(integer) 0

redis> APPEND myphone "nokia"       # 对不存在的 key 进行 APPEND ，等同于 SET myphone "nokia"
(integer) 5                         # 字符长度


# 对已存在的字符串进行 APPEND

redis> APPEND myphone " - 1110"     # 长度从 5 个字符增加到 12 个字符
(integer) 12

redis> GET myphone
"nokia - 1110"
```

#### 模式：时间序列(Time series)

　　`APPEND` 可以为一系列定长(fixed-size)数据(sample)提供一种紧凑的表示方式，通常称之为时间序列。

　　每当一个新数据到达的时候，执行以下命令：

```bash
APPEND timeseries "fixed-size sample"
```

　　然后可以通过以下的方式访问时间序列的各项属性：

- `STRLEN` 给出时间序列中数据的数量
- `GETRANGE` 可以用于随机访问。只要有相关的时间信息的话，我们就可以在 Redis 2.6 中使用 Lua 脚本和 `GETRANGE` 命令实现二分查找。
- `SETRANGE` 可以用于覆盖或修改已存在的的时间序列。

　　这个模式的唯一缺陷是我们只能增长时间序列，而不能对时间序列进行缩短，因为 Redis 目前还没有对字符串进行修剪(tirm)的命令，但是，不管怎么说，这个模式的储存方式还是可以节省下大量的空间。

　　可以考虑使用 UNIX 时间戳作为时间序列的键名，这样一来，可以避免单个 `key` 因为保存过大的时间序列而占用大量内存，另一方面，也可以节省下大量命名空间。

　　下面是一个时间序列的例子：

```bash
redis> APPEND ts "0043"
(integer) 4

redis> APPEND ts "0035"
(integer) 8

redis> GETRANGE ts 0 3
"0043"

redis> GETRANGE ts 4 7
"0035"
```

### DECR

　　**DECR key**

　　将 `key` 中储存的数字值减一。

　　如果 `key` 不存在，那么 `key` 的值会先被初始化为 `0` ，然后再执行 `DECR` 操作。

　　如果值包含错误的类型，或字符串类型的值不能表示为数字，那么返回一个错误。

　　本操作的值限制在 64 位(bit)有符号数字表示之内。

　　关于递增(increment) / 递减(decrement)操作的更多信息，请参见 `INCR` 命令。

　　**可用版本：** >= 1.0.0

　　**时间复杂度：**
O(1)

　　**返回值：**
执行 `DECR` 命令之后 `key` 的值。

```bash
# 对存在的数字值 key 进行 DECR

redis> SET failure_times 10
OK

redis> DECR failure_times
(integer) 9


# 对不存在的 key 值进行 DECR

redis> EXISTS count
(integer) 0

redis> DECR count
(integer) -1


# 对存在但不是数值的 key 进行 DECR

redis> SET company YOUR_CODE_SUCKS.LLC
OK

redis> DECR company
(error) ERR value is not an integer or out of range
```

### DECRBY

　　**DECRBY key decrement**

　　将 `key` 所储存的值减去减量 `decrement` 。

　　如果 `key` 不存在，那么 `key` 的值会先被初始化为 `0` ，然后再执行 `DECRBY` 操作。

　　如果值包含错误的类型，或字符串类型的值不能表示为数字，那么返回一个错误。

　　本操作的值限制在 64 位(bit)有符号数字表示之内。

　　关于更多递增(increment) / 递减(decrement)操作的更多信息，请参见 `INCR` 命令。

　　**可用版本：** >= 1.0.0

　　**时间复杂度：**
O(1)

　　**返回值：**
减去 `decrement` 之后， `key` 的值。

```
# 对已存在的 key 进行 DECRBY

redis> SET count 100
OK

redis> DECRBY count 20
(integer) 80


# 对不存在的 key 进行DECRBY

redis> EXISTS pages
(integer) 0

redis> DECRBY pages 10
(integer) -10
```

### GET

　　**GET key**

　　返回 `key` 所关联的字符串值。

　　如果 `key` 不存在那么返回特殊值 `nil` 。

　　假如 `key` 储存的值不是字符串类型，返回一个错误，因为 `GET` 只能用于处理字符串值。

　　**可用版本：** >= 1.0.0

　　**时间复杂度：**
O(1)

　　**返回值：**
| 当 `key` 不存在时，返回 `nil` ，否则，返回 `key` 的值。
| 如果 `key` 不是字符串类型，那么返回一个错误。

```
# 对不存在的 key 或字符串类型 key 进行 GET

redis> GET db
(nil)

redis> SET db redis
OK

redis> GET db
"redis"


# 对不是字符串类型的 key 进行 GET

redis> DEL db
(integer) 1

redis> LPUSH db redis mongodb mysql
(integer) 3

redis> GET db
(error) ERR Operation against a key holding the wrong kind of value
```

### GETRANGE

　　**GETRANGE key start end**

　　返回 `key` 中字符串值的子字符串，字符串的截取范围由 `start` 和 `end` 两个偏移量决定(包括 `start` 和 `end` 在内)。

　　负数偏移量表示从字符串最后开始计数， `-1` 表示最后一个字符， `-2` 表示倒数第二个，以此类推。

　　`GETRANGE` 通过保证子字符串的值域(range)不超过实际字符串的值域来处理超出范围的值域请求。

　　在 <= 2.0 的版本里，GETRANGE 被叫作 SUBSTR。

　　**可用版本：** >= 2.4.0

　　**时间复杂度：**
| O(N)， `N` 为要返回的字符串的长度。
| 复杂度最终由字符串的返回值长度决定，但因为从已有字符串中取出子字符串的操作非常廉价(cheap)，所以对于长度不大的字符串，该操作的复杂度也可看作 O(1)。

　　**返回值：**
截取得出的子字符串。

```
redis> SET greeting "hello, my friend"
OK

redis> GETRANGE greeting 0 4          # 返回索引0-4的字符，包括4。
"hello"

redis> GETRANGE greeting -1 -5        # 不支持回绕操作
""

redis> GETRANGE greeting -3 -1        # 负数索引
"end"

redis> GETRANGE greeting 0 -1         # 从第一个到最后一个
"hello, my friend"

redis> GETRANGE greeting 0 1008611    # 值域范围不超过实际字符串，超过部分自动被符略
"hello, my friend"
```

### GETSET

　　**GETSET key value**

　　将给定 `key` 的值设为 `value` ，并返回 `key` 的旧值(old value)。

　　当 `key` 存在但不是字符串类型时，返回一个错误。

　　**可用版本：** >= 1.0.0

　　**时间复杂度：**
O(1)

　　**返回值：**
返回给定 `key` 的旧值。
当 `key` 没有旧值时，也即是， `key` 不存在时，返回 `nil` 。

```
redis> GETSET db mongodb    # 没有旧值，返回 nil
(nil)

redis> GET db
"mongodb"

redis> GETSET db redis      # 返回旧值 mongodb
"mongodb"

redis> GET db
"redis"
```

#### 模式

　　`GETSET` 可以和 `INCR` 组合使用，实现一个有原子性(atomic)复位操作的计数器(counter)。

　　举例来说，每次当某个事件发生时，进程可能对一个名为 `mycount` 的 `key` 调用 `INCR` 操作，通常我们还要在一个原子时间内同时完成获得计数器的值和将计数器值复位为 `0` 两个操作。

　　可以用命令 `GETSET mycounter 0` 来实现这一目标。

```
redis> INCR mycount
(integer) 11

redis> GETSET mycount 0  # 一个原子内完成 GET mycount 和 SET mycount 0 操作
"11"

redis> GET mycount       # 计数器被重置
"0"
```

### INCR

　　**INCR key**

　　将 `key` 中储存的数字值增一。

　　如果 `key` 不存在，那么 `key` 的值会先被初始化为 `0` ，然后再执行 `INCR` 操作。

　　如果值包含错误的类型，或字符串类型的值不能表示为数字，那么返回一个错误。

　　本操作的值限制在 64 位(bit)有符号数字表示之内。

　　这是一个针对字符串的操作，因为 Redis 没有专用的整数类型，所以 key 内储存的字符串被解释为十进制 64 位有符号整数来执行 INCR 操作。

　　**可用版本：** >= 1.0.0

　　**时间复杂度：**
O(1)

　　**返回值：**
执行 `INCR` 命令之后 `key` 的值。

```
redis> SET page_view 20
OK

redis> INCR page_view
(integer) 21

redis> GET page_view    # 数字值在 Redis 中以字符串的形式保存
"21"
```

　　**模式**：计数器

　　计数器是 Redis 的原子性自增操作可实现的最直观的模式了，它的想法相当简单：每当某个操作发生时，向 Redis 发送一个 `INCR` 命令。

　　比如在一个 web 应用程序中，如果想知道用户在一年中每天的点击量，那么只要将用户 ID 以及相关的日期信息作为键，并在每次用户点击页面时，执行一次自增操作即可。

　　比如用户名是 `peter` ，点击时间是 2012 年 3 月 22 日，那么执行命令 `INCR peter::2012.3.22` 。

　　可以用以下几种方式扩展这个简单的模式：

- 可以通过组合使用 `INCR` 和 `expire` ，来达到只在规定的生存时间内进行计数(counting)的目的。
- 客户端可以通过使用 `GETSET` 命令原子性地获取计数器的当前值并将计数器清零，更多信息请参考 `GETSET` 命令。
- 使用其他自增/自减操作，比如 `DECR` 和 `INCRBY` ，用户可以通过执行不同的操作增加或减少计数器的值，比如在游戏中的记分器就可能用到这些命令。

#### 模式：限速器

　　限速器是特殊化的计算器，它用于限制一个操作可以被执行的速率(rate)。

　　限速器的典型用法是限制公开 API 的请求次数，以下是一个限速器实现示例，它将 API 的最大请求数限制在每个 IP 地址每秒钟十个之内：

```
FUNCTION LIMIT_API_CALL(ip)
ts = CURRENT_UNIX_TIME()
keyname = ip+":"+ts
current = GET(keyname)

IF current != NULL AND current > 10 THEN
    ERROR "too many requests per second"
END

IF current == NULL THEN
    MULTI
        INCR(keyname, 1)
        EXPIRE(keyname, 1)
    EXEC
ELSE
    INCR(keyname, 1)
END

PERFORM_API_CALL()
```

　　这个实现每秒钟为每个 IP 地址使用一个不同的计数器，并用 `EXPIRE` 命令设置生存时间(这样 Redis 就会负责自动删除过期的计数器)。

　　注意，我们使用事务打包执行 `INCR` 命令和 `EXPIRE` 命令，避免引入竞争条件，保证每次调用 API 时都可以正确地对计数器进行自增操作并设置生存时间。

　　以下是另一个限速器实现：

```
FUNCTION LIMIT_API_CALL(ip):
current = GET(ip)
IF current != NULL AND current > 10 THEN
    ERROR "too many requests per second"
ELSE
    value = INCR(ip)
    IF value == 1 THEN
        EXPIRE(ip,1)
    END
    PERFORM_API_CALL()
END
```

　　这个限速器只使用单个计数器，它的生存时间为一秒钟，如果在一秒钟内，这个计数器的值大于 `10` 的话，那么访问就会被禁止。

　　这个新的限速器在思路方面是没有问题的，但它在实现方面不够严谨，如果我们仔细观察一下的话，就会发现在 `INCR` 和 `EXPIRE` 之间存在着一个竞争条件，假如客户端在执行 `INCR` 之后，因为某些原因(比如客户端失败)而忘记设置 `EXPIRE` 的话，那么这个计数器就会一直存在下去，造成每个用户只能访问 `10` 次，噢，这简直是个灾难！

　　要消灭这个实现中的竞争条件，我们可以将它转化为一个 Lua 脚本，并放到 Redis 中运行(这个方法仅限于 Redis 2.6 及以上的版本)：

```
local current
current = redis.call("incr",KEYS[1])
if tonumber(current) == 1 then
    redis.call("expire",KEYS[1],1)
end
```

　　通过将计数器作为脚本放到 Redis 上运行，我们保证了 `INCR` 和 `EXPIRE` 两个操作的原子性，现在这个脚本实现不会引入竞争条件，它可以运作的很好。

　　关于在 Redis 中运行 Lua 脚本的更多信息，请参考 `EVAL` 命令。

　　还有另一种消灭竞争条件的方法，就是使用 Redis 的列表结构来代替 `INCR` 命令，这个方法无须脚本支持，因此它在 Redis 2.6 以下的版本也可以运行得很好：

```
FUNCTION LIMIT_API_CALL(ip)
current = LLEN(ip)
IF current > 10 THEN
    ERROR "too many requests per second"
ELSE
    IF EXISTS(ip) == FALSE
        MULTI
            RPUSH(ip,ip)
            EXPIRE(ip,1)
        EXEC
    ELSE
        RPUSHX(ip,ip)
    END
    PERFORM_API_CALL()
END
```

　　新的限速器使用了列表结构作为容器， `LLEN` 用于对访问次数进行检查，一个事务包裹着 `RPUSH` 和 `EXPIRE` 两个命令，用于在第一次执行计数时创建列表，并正确设置地设置过期时间，最后， `RPUSHX` 在后续的计数操作中进行增加操作。

### INCRBY

　　**INCRBY key increment**

　　将 `key` 所储存的值加上增量 `increment` 。

　　如果 `key` 不存在，那么 `key` 的值会先被初始化为 `0` ，然后再执行 `INCRBY` 命令。

　　如果值包含错误的类型，或字符串类型的值不能表示为数字，那么返回一个错误。

　　本操作的值限制在 64 位(bit)有符号数字表示之内。

　　关于递增(increment) / 递减(decrement)操作的更多信息，参见 `INCR` 命令。

　　**可用版本：** >= 1.0.0

　　**时间复杂度：**
O(1)

　　**返回值：**
加上 `increment` 之后， `key` 的值。

```
# key 存在且是数字值

redis> SET rank 50
OK

redis> INCRBY rank 20
(integer) 70

redis> GET rank
"70"


# key 不存在时

redis> EXISTS counter
(integer) 0

redis> INCRBY counter 30
(integer) 30

redis> GET counter
"30"


# key 不是数字值时

redis> SET book "long long ago..."
OK

redis> INCRBY book 200
(error) ERR value is not an integer or out of range
```

### INCRBYFLOAT

　　**INCRBYFLOAT key increment**

　　为 `key` 中所储存的值加上浮点数增量 `increment` 。

　　如果 `key` 不存在，那么 `INCRBYFLOAT` 会先将 `key` 的值设为 `0` ，再执行加法操作。

　　如果命令执行成功，那么 `key` 的值会被更新为（执行加法之后的）新值，并且新值会以字符串的形式返回给调用者。

　　无论是 `key` 的值，还是增量 `increment` ，都可以使用像 `2.0e7` 、 `3e5` 、 `90e-2` 那样的指数符号(exponential notation)来表示，但是，\ **执行 INCRBYFLOAT 命令之后的值**\ 总是以同样的形式储存，也即是，它们总是由一个数字，一个（可选的）小数点和一个任意位的小数部分组成（比如 `3.14` 、 `69.768` ，诸如此类)，小数部分尾随的 `0` 会被移除，如果有需要的话，还会将浮点数改为整数（比如 `3.0` 会被保存成 `3` ）。

　　除此之外，无论加法计算所得的浮点数的实际精度有多长， `INCRBYFLOAT` 的计算结果也最多只能表示小数点的后十七位。

　　当以下任意一个条件发生时，返回一个错误：

- `key` 的值不是字符串类型(因为 Redis 中的数字和浮点数都以字符串的形式保存，所以它们都属于字符串类型）
- `key` 当前的值或者给定的增量 `increment` 不能解释(parse)为双精度浮点数(double precision floating point number）

　　**可用版本：** >= 2.6.0

　　**时间复杂度：**
O(1)

　　**返回值：**
执行命令之后 `key` 的值。

```
# 值和增量都不是指数符号

redis> SET mykey 10.50
OK

redis> INCRBYFLOAT mykey 0.1
"10.6"


# 值和增量都是指数符号

redis> SET mykey 314e-2
OK

redis> GET mykey                # 用 SET 设置的值可以是指数符号
"314e-2"

redis> INCRBYFLOAT mykey 0      # 但执行 INCRBYFLOAT 之后格式会被改成非指数符号
"3.14"


# 可以对整数类型执行

redis> SET mykey 3
OK

redis> INCRBYFLOAT mykey 1.1
"4.1"


# 后跟的 0 会被移除

redis> SET mykey 3.0
OK

redis> GET mykey                                    # SET 设置的值小数部分可以是 0
"3.0"

redis> INCRBYFLOAT mykey 1.000000000000000000000    # 但 INCRBYFLOAT 会将无用的 0 忽略掉，有需要的话，将浮点变为整数
"4"

redis> GET mykey
"4"
```

### MGET

　　**MGET key [key ...]**

　　返回所有(一个或多个)给定 `key` 的值。

　　如果给定的 `key` 里面，有某个 `key` 不存在，那么这个 `key` 返回特殊值 `nil` 。因此，该命令永不失败。

　　**可用版本：** >= 1.0.0

　　**时间复杂度:**
O(N) , `N` 为给定 `key` 的数量。

　　**返回值：**
一个包含所有给定 `key` 的值的列表。

```
redis> SET redis redis.com
OK

redis> SET mongodb mongodb.org
OK

redis> MGET redis mongodb
1) "redis.com"
2) "mongodb.org"

redis> MGET redis mongodb mysql     # 不存在的 mysql 返回 nil
1) "redis.com"
2) "mongodb.org"
3) (nil)
```

### MSET

　　**MSET key value [key value ...]**

　　同时设置一个或多个 `key-value` 对。

　　如果某个给定 `key` 已经存在，那么 `MSET` 会用新值覆盖原来的旧值，如果这不是你所希望的效果，请考虑使用 `MSETNX` 命令：它只会在所有给定 `key` 都不存在的情况下进行设置操作。

　　`MSET` 是一个原子性(atomic)操作，所有给定 `key` 都会在同一时间内被设置，某些给定 `key` 被更新而另一些给定 `key` 没有改变的情况，不可能发生。

　　**可用版本：** >= 1.0.1

　　**时间复杂度：**
O(N)， `N` 为要设置的 `key` 数量。

　　**返回值：**
总是返回 `OK` (因为 `MSET` 不可能失败)

```
redis> MSET date "2012.3.30" time "11:00 a.m." weather "sunny"
OK

redis> MGET date time weather
1) "2012.3.30"
2) "11:00 a.m."
3) "sunny"


# MSET 覆盖旧值例子

redis> SET google "google.hk"
OK

redis> MSET google "google.com"
OK

redis> GET google
"google.com"
```

### MSETNX

　　**MSETNX key value [key value ...]**

　　同时设置一个或多个 `key-value` 对，当且仅当所有给定 `key` 都不存在。

　　即使只有一个给定 `key` 已存在， `MSETNX` 也会拒绝执行所有给定 `key` 的设置操作。

　　`MSETNX` 是原子性的，因此它可以用作设置多个不同 `key` 表示不同字段(field)的唯一性逻辑对象(unique logic object)，所有字段要么全被设置，要么全不被设置。

　　**可用版本：** >= 1.0.1

　　**时间复杂度：**
O(N)， `N` 为要设置的 `key` 的数量。

　　**返回值：**
| 当所有 `key` 都成功设置，返回 `1` 。
| 如果所有给定 `key` 都设置失败(至少有一个 `key` 已经存在)，那么返回 `0` 。

```
# 对不存在的 key 进行 MSETNX

redis> MSETNX rmdbs "MySQL" nosql "MongoDB" key-value-store "redis"
(integer) 1

redis> MGET rmdbs nosql key-value-store
1) "MySQL"
2) "MongoDB"
3) "redis"


# MSET 的给定 key 当中有已存在的 key

redis> MSETNX rmdbs "Sqlite" language "python"  # rmdbs 键已经存在，操作失败
(integer) 0

redis> EXISTS language                          # 因为 MSET 是原子性操作，language 没有被设置
(integer) 0

redis> GET rmdbs                                # rmdbs 也没有被修改
"MySQL"
```

### PSETEX

　　**PSETEX key milliseconds value**

　　这个命令和 `SETEX` 命令相似，但它以毫秒为单位设置 `key` 的生存时间，而不是像 `SETEX` 命令那样，以秒为单位。

　　**可用版本：** >= 2.6.0

　　**时间复杂度：**
O(1)

　　**返回值：**
设置成功时返回 `OK` 。

```
redis> PSETEX mykey 1000 "Hello"
OK

redis> PTTL mykey
(integer) 999

redis> GET mykey
"Hello"
```

### SET

　　**SET key value [EX seconds][px milliseconds] [NX|XX]**

　　将字符串值 `value` 关联到 `key` 。

　　如果 `key` 已经持有其他值， `SET` 就覆写旧值，无视类型。

　　对于某个原本带有生存时间（TTL）的键来说，
当 `SET` 命令成功在这个键上执行时，
这个键原有的 TTL 将被清除。

　　**可选参数**

　　从 Redis 2.6.12 版本开始， `SET` 命令的行为可以通过一系列参数来修改：

- `EX second` ：设置键的过期时间为 `second` 秒。 `SET key value EX second` 效果等同于 `SETEX key second value` 。
- `PX millisecond` ：设置键的过期时间为 `millisecond` 毫秒。 `SET key value PX millisecond` 效果等同于 `PSETEX key millisecond value` 。
- `NX` ：只在键不存在时，才对键进行设置操作。 `SET key value NX` 效果等同于 `SETNX key value` 。
- `XX` ：只在键已经存在时，才对键进行设置操作。

　　因为 `SET` 命令可以通过参数来实现和 `SETNX` 、 `SETEX` 和 `PSETEX` 三个命令的效果，所以将来的 Redis 版本可能会废弃并最终移除 `SETNX` 、 `SETEX` 和 `PSETEX` 这三个命令。

　　**可用版本：** >= 1.0.0

　　**时间复杂度：**
O(1)

　　**返回值：**
在 Redis 2.6.12 版本以前， `SET` 命令总是返回 `OK` 。

　　从 Redis 2.6.12 版本开始， `SET` 在设置操作成功完成时，才返回 `OK` 。
如果设置了 `NX` 或者 `XX` ，但因为条件没达到而造成设置操作未执行，那么命令返回空批量回复（NULL Bulk Reply）。

```
# 对不存在的键进行设置

redis 127.0.0.1:6379> SET key "value"
OK

redis 127.0.0.1:6379> GET key
"value"


# 对已存在的键进行设置

redis 127.0.0.1:6379> SET key "new-value"
OK

redis 127.0.0.1:6379> GET key
"new-value"


# 使用 EX 选项

redis 127.0.0.1:6379> SET key-with-expire-time "hello" EX 10086
OK

redis 127.0.0.1:6379> GET key-with-expire-time
"hello"

redis 127.0.0.1:6379> TTL key-with-expire-time
(integer) 10069


# 使用 PX 选项

redis 127.0.0.1:6379> SET key-with-pexpire-time "moto" PX 123321
OK

redis 127.0.0.1:6379> GET key-with-pexpire-time
"moto"

redis 127.0.0.1:6379> PTTL key-with-pexpire-time
(integer) 111939


# 使用 NX 选项

redis 127.0.0.1:6379> SET not-exists-key "value" NX
OK      # 键不存在，设置成功

redis 127.0.0.1:6379> GET not-exists-key
"value"

redis 127.0.0.1:6379> SET not-exists-key "new-value" NX
(nil)   # 键已经存在，设置失败

redis 127.0.0.1:6379> GEt not-exists-key
"value" # 维持原值不变


# 使用 XX 选项

redis 127.0.0.1:6379> EXISTS exists-key
(integer) 0

redis 127.0.0.1:6379> SET exists-key "value" XX
(nil)   # 因为键不存在，设置失败

redis 127.0.0.1:6379> SET exists-key "value"
OK      # 先给键设置一个值

redis 127.0.0.1:6379> SET exists-key "new-value" XX
OK      # 设置新值成功

redis 127.0.0.1:6379> GET exists-key
"new-value"


# NX 或 XX 可以和 EX 或者 PX 组合使用

redis 127.0.0.1:6379> SET key-with-expire-and-NX "hello" EX 10086 NX
OK

redis 127.0.0.1:6379> GET key-with-expire-and-NX
"hello"

redis 127.0.0.1:6379> TTL key-with-expire-and-NX
(integer) 10063

redis 127.0.0.1:6379> SET key-with-pexpire-and-XX "old value"
OK

redis 127.0.0.1:6379> SET key-with-pexpire-and-XX "new value" PX 123321
OK

redis 127.0.0.1:6379> GET key-with-pexpire-and-XX
"new value"

redis 127.0.0.1:6379> PTTL key-with-pexpire-and-XX
(integer) 112999


# EX 和 PX 可以同时出现，但后面给出的选项会覆盖前面给出的选项

redis 127.0.0.1:6379> SET key "value" EX 1000 PX 5000000
OK

redis 127.0.0.1:6379> TTL key
(integer) 4993  # 这是 PX 参数设置的值

redis 127.0.0.1:6379> SET another-key "value" PX 5000000 EX 1000
OK

redis 127.0.0.1:6379> TTL another-key
(integer) 997   # 这是 EX 参数设置的值
```

　　**使用模式**

　　命令 `SET resource-name anystring NX EX max-lock-time` 是一种在 Redis 中实现锁的简单方法。

　　客户端执行以上的命令：

- 如果服务器返回 `OK` ，那么这个客户端获得锁。
- 如果服务器返回 `NIL` ，那么客户端获取锁失败，可以在稍后再重试。

　　设置的过期时间到达之后，锁将自动释放。

　　可以通过以下修改，让这个锁实现更健壮：

- 不使用固定的字符串作为键的值，而是设置一个不可猜测（non-guessable）的长随机字符串，作为口令串（token）。
- 不使用 `DEL` 命令来释放锁，而是发送一个 Lua 脚本，这个脚本只在客户端传入的值和键的口令串相匹配时，才对键进行删除。

　　这两个改动可以防止持有过期锁的客户端误删现有锁的情况出现。

　　以下是一个简单的解锁脚本示例：

```lua
if redis.call("get",KEYS[1]) == ARGV[1]
    then
        return redis.call("del",KEYS[1])
    else
        return 0
    end
```

　　这个脚本可以通过 `EVAL ...script... 1 resource-name token-value` 命令来调用。

### SETEX

　　**SETEX key seconds value**

　　将值 `value` 关联到 `key` ，并将 `key` 的生存时间设为 `seconds` (以秒为单位)。

　　如果 `key` \ 已经存在， `SETEX` 命令将覆写旧值。

　　这个命令类似于以下两个命令：

```
SET key value
EXPIRE key seconds  # 设置生存时间
```

　　不同之处是， `SETEX` 是一个原子性(atomic)操作，关联值和设置生存时间两个动作会在同一时间内完成，该命令在 Redis 用作缓存时，非常实用。

　　**可用版本：** >= 2.0.0

　　**时间复杂度：**
O(1)

　　**返回值：**
| 设置成功时返回 `OK` 。
| 当 `seconds` 参数不合法时，返回一个错误。

```
# 在 key 不存在时进行 SETEX

redis> SETEX cache_user_id 60 10086
OK

redis> GET cache_user_id  # 值
"10086"

redis> TTL cache_user_id  # 剩余生存时间
(integer) 49


# key 已经存在时，SETEX 覆盖旧值

redis> SET cd "timeless"
OK

redis> SETEX cd 3000 "goodbye my love"
OK

redis> GET cd
"goodbye my love"

redis> TTL cd
(integer) 2997
```

### SETNX

　　**SETNX key value**

　　将 `key` 的值设为 `value` ，当且仅当 `key` 不存在。

　　若给定的 `key` 已经存在，则 `SETNX` 不做任何动作。

　　`SETNX` 是『SET if Not eXists』(如果不存在，则 SET)的简写。

　　**可用版本：** >= 1.0.0

　　**时间复杂度：**
O(1)

　　**返回值：**
| 设置成功，返回 `1` 。
| 设置失败，返回 `0` 。

```
redis> EXISTS job                # job 不存在
(integer) 0

redis> SETNX job "programmer"    # job 设置成功
(integer) 1

redis> SETNX job "code-farmer"   # 尝试覆盖 job ，失败
(integer) 0

redis> GET job                   # 没有被覆盖
"programmer"
```

#### 模式：将 SETNX 用于加锁(locking)

　　.. warning:: 已经证实这个加锁算法带有竞争条件，在特定情况下会造成错误，请不要使用这个加锁算法，具体请参考 `http://huangz.iteye.com/blog/1381538 <http://huangz.iteye.com/blog/1381538>`_ 。
`SETNX`_ 可以用作加锁原语(locking primitive)。比如说，要对关键字(key) `foo` 加锁，客户端可以尝试以下方式：
`SETNX lock.foo <current Unix time + lock timeout + 1>`
如果 `SETNX`_ 返回 `1` ，说明客户端已经获得了锁， `key` 设置的 unix 时间则指定了锁失效的时间。之后客户端可以通过 `DEL lock.foo` 来释放锁。
如果 `SETNX`_ 返回 `0` ，说明 `key` 已经被其他客户端上锁了。如果锁是非阻塞(non blocking lock)的，我们可以选择返回调用，或者进入一个重试循环，直到成功获得锁或重试超时(timeout)。
**处理死锁(deadlock)**
上面的锁算法有一个问题：如果因为客户端失败、崩溃或其他原因导致没有办法释放锁的话，怎么办？
这种状况可以通过检测发现——因为上锁的 `key` 保存的是 unix 时间戳，假如 `key` 值的时间戳小于当前的时间戳，表示锁已经不再有效。
但是，当有多个客户端同时检测一个锁是否过期并尝试释放它的时候，我们不能简单粗暴地删除死锁的 `key` ，再用 `SETNX`_ 上锁，因为这时竞争条件(race condition)已经形成了： * C1 和 C2 读取 *`_lock.foo_`* 并检查时间戳， _`_SETNX_` 都返回 `0` ，因为它已经被 C3 锁上了，但 C3 在上锁之后就崩溃(crashed)了。
_ C1 向 `lock.foo` 发送 `DEL` 命令。
_ C1 向 `lock.foo` 发送 `SETNX` 并成功。

- C2 向 `lock.foo` 发送 `del` 命令。
- C2 向 `lock.foo` 发送 `SETNX`_ 并成功。
  _ 出错：因为竞争条件的关系，C1 和 C2 两个都获得了锁。
  幸好，以下算法可以避免以上问题。来看看我们聪明的 C4 客户端怎么办：
  _ C4 向 `lock.foo` 发送 `SETNX` 命令。
  _ 因为崩溃掉的 C3 还锁着 `lock.foo` ，所以 Redis 向 C4 返回 `0` 。
  _ C4 向 `lock.foo` 发送 `GET` 命令，查看 `lock.foo` 的锁是否过期。如果不，则休眠(sleep)一段时间，并在之后重试。
  _ 另一方面，如果 `lock.foo` 内的 unix 时间戳比当前时间戳老，C4 执行以下命令：
  `GETSET lock.foo <current Unix timestamp + lock timeout + 1>`
  因为 `GETSET` 的作用，C4 可以检查看 `GETSET` 的返回值，确定 `lock.foo` 之前储存的旧值仍是那个过期时间戳，如果是的话，那么 C4 获得锁。 * 如果其他客户端，比如 C5，比 C4 更快地执行了 `GETSET` 操作并获得锁，那么 C4 的 `GETSET` 操作返回的就是一个未过期的时间戳(C5 设置的时间戳)。C4 只好从第一步开始重试。
  | 注意，即便 C4 的 `GETSET` 操作对 `key` 进行了修改，这对未来也没什么影响。
  .. warning:: 为了让这个加锁算法更健壮，获得锁的客户端应该常常检查过期时间以免锁因诸如 `DEL` 等命令的执行而被意外解开，因为客户端失败的情况非常复杂，不仅仅是崩溃这么简单，还可能是客户端因为某些操作被阻塞了相当长时间，紧接着 `DEL` 命令被尝试执行(但这时锁却在另外的客户端手上)。

### SETRANGE

　　**SETRANGE key offset value**

　　用 `value` 参数覆写(overwrite)给定 `key` 所储存的字符串值，从偏移量 `offset` 开始。

　　不存在的 `key` 当作空白字符串处理。

　　`SETRANGE` 命令会确保字符串足够长以便将 `value` 设置在指定的偏移量上，如果给定 `key` 原来储存的字符串长度比偏移量小(比如字符串只有 `5` 个字符长，但你设置的 `offset` 是 `10` )，那么原字符和偏移量之间的空白将用零字节(zerobytes, `"\x00"` )来填充。

　　注意你能使用的最大偏移量是 2^29-1(536870911) ，因为 Redis 字符串的大小被限制在 512 兆(megabytes)以内。如果你需要使用比这更大的空间，你可以使用多个 `key` 。

> warning::
> 当生成一个很长的字符串时，Redis 需要分配内存空间，该操作有时候可能会造成服务器阻塞(block)。在 2010 年的 Macbook Pro 上，设置偏移量为 536870911(512MB 内存分配)，耗费约 300 毫秒，
> 设置偏移量为 134217728(128MB 内存分配)，耗费约 80 毫秒，设置偏移量 33554432(32MB 内存分配)，耗费约 30 毫秒，设置偏移量为 8388608(8MB 内存分配)，耗费约 8 毫秒。
> 注意若首次内存分配成功之后，再对同一个 `key` 调用 `SETRANGE` 操作，无须再重新内存。

　　**可用版本：** >= 2.2.0

　　**时间复杂度：**
| 对小(small)的字符串，平摊复杂度 O(1)。(关于什么字符串是"小"的，请参考 `APPEND` 命令)
| 否则为 O(M)， `M` 为 `value` 参数的长度。

　　**返回值：**
被 `SETRANGE` 修改之后，字符串的长度。

```
# 对非空字符串进行 SETRANGE

redis> SET greeting "hello world"
OK

redis> SETRANGE greeting 6 "Redis"
(integer) 11

redis> GET greeting
"hello Redis"


# 对空字符串/不存在的 key 进行 SETRANGE

redis> EXISTS empty_string
(integer) 0

redis> SETRANGE empty_string 5 "Redis!"   # 对不存在的 key 使用 SETRANGE
(integer) 11

redis> GET empty_string                   # 空白处被"\x00"填充
"\x00\x00\x00\x00\x00Redis!"
```

#### 模式

　　因为有了 `SETRANGE` 和 `GETRANGE` 命令，你可以将 Redis 字符串用作具有 O(1)随机访问时间的线性数组，这在很多真实用例中都是非常快速且高效的储存方式，具体请参考 `APPEND` 命令的『模式：时间序列』部分。

### STRLEN

　　**STRLEN key**

　　返回 `key` 所储存的字符串值的长度。

　　当 `key` 储存的不是字符串值时，返回一个错误。

　　**可用版本：** >= 2.2.0

　　**复杂度：**
O(1)

　　**返回值：**
| 字符串值的长度。
| 当 `key` 不存在时，返回 `0` 。

```
# 获取字符串的长度

redis> SET mykey "Hello world"
OK

redis> STRLEN mykey
(integer) 11


# 不存在的 key 长度为 0

redis> STRLEN nonexisting
(integer) 0
```

## Hash（哈希表）

　　压缩链表ziplist
Hashtable类似HashMap都采用链表+数组

- 当同时满足下面两个条件使用ziplist编码，否则使用hashtable编码
  - 列表保存元素个数小于512个
  - 每个元素长度小于64字节

### DEL

　　**HDEL key field [field ...]**

　　删除哈希表 `key` 中的一个或多个指定域，不存在的域将被忽略。

　　在 Redis2.4 以下的版本里， `HDEL` 每次只能删除单个域，如果你需要在一个原子时间内删除多个域，请将命令包含在 `MULTI` / `EXEC` 块内。

　　**可用版本：** >= 2.0.0

　　**时间复杂度:**
O(N)， `N` 为要删除的域的数量。

　　**返回值:**
被成功移除的域的数量，不包括被忽略的域。

```
# 测试数据

redis> HGETALL abbr
1) "a"
2) "apple"
3) "b"
4) "banana"
5) "c"
6) "cat"
7) "d"
8) "dog"


# 删除单个域

redis> HDEL abbr a
(integer) 1


# 删除不存在的域

redis> HDEL abbr not-exists-field
(integer) 0


# 删除多个域

redis> HDEL abbr b c
(integer) 2

redis> HGETALL abbr
1) "d"
2) "dog"
```

### HEXISTS

　　**HEXISTS key field**

　　查看哈希表 `key` 中，给定域 `field` 是否存在。

　　**可用版本：** >= 2.0.0

　　**时间复杂度：**
O(1)

　　**返回值：**
| 如果哈希表含有给定域，返回 `1` 。
| 如果哈希表不含有给定域，或 `key` 不存在，返回 `0` 。

　　::

```
redis> HEXISTS phone myphone
(integer) 0

redis> HSET phone myphone nokia-1110
(integer) 1

redis> HEXISTS phone myphone
(integer) 1
```

### HGET

　　**HGET key field**

　　返回哈希表 `key` 中给定域 `field` 的值。

　　**可用版本：** >= 2.0.0

　　**时间复杂度：**
O(1)

　　**返回值：**
| 给定域的值。
| 当给定域不存在或是给定 `key` 不存在时，返回 `nil` 。

　　::

```
# 域存在

redis> HSET site redis redis.com
(integer) 1

redis> HGET site redis
"redis.com"


# 域不存在

redis> HGET site mysql
(nil)
```

### HGETALL

　　**HGETALL key**

　　返回哈希表 `key` 中，所有的域和值。

　　在返回值里，紧跟每个域名(field name)之后是域的值(value)，所以返回值的长度是哈希表大小的两倍。

　　**可用版本：** >= 2.0.0

　　**时间复杂度：**
O(N)， `N` 为哈希表的大小。

　　**返回值：**
| 以列表形式返回哈希表的域和域的值。
| 若 `key` 不存在，返回空列表。

　　::

```
redis> HSET people jack "Jack Sparrow"
(integer) 1

redis> HSET people gump "Forrest Gump"
(integer) 1

redis> HGETALL people
1) "jack"          # 域
2) "Jack Sparrow"  # 值
3) "gump"
4) "Forrest Gump"
```

### HINCRBY

　　**HINCRBY key field increment**

　　为哈希表 `key` 中的域 `field` 的值加上增量 `increment` 。

　　增量也可以为负数，相当于对给定域进行减法操作。

　　如果 `key` 不存在，一个新的哈希表被创建并执行 `HINCRBY` 命令。

　　如果域 `field` 不存在，那么在执行命令前，域的值被初始化为 `0` 。

　　对一个储存字符串值的域 `field` 执行 `HINCRBY` 命令将造成一个错误。

　　本操作的值被限制在 64 位(bit)有符号数字表示之内。

　　**可用版本：** >= 2.0.0

　　**时间复杂度：**
O(1)

　　**返回值：**
执行 `HINCRBY` 命令之后，哈希表 `key` 中域 `field` 的值。

　　::

```
# increment 为正数

redis> HEXISTS counter page_view    # 对空域进行设置
(integer) 0

redis> HINCRBY counter page_view 200
(integer) 200

redis> HGET counter page_view
"200"


# increment 为负数

redis> HGET counter page_view
"200"

redis> HINCRBY counter page_view -50
(integer) 150

redis> HGET counter page_view
"150"


# 尝试对字符串值的域执行HINCRBY命令

redis> HSET myhash string hello,world       # 设定一个字符串值
(integer) 1

redis> HGET myhash string
"hello,world"

redis> HINCRBY myhash string 1              # 命令执行失败，错误。
(error) ERR hash value is not an integer

redis> HGET myhash string                   # 原值不变
"hello,world"
```

### HINCRBYFLOAT

　　**HINCRBYFLOAT key field increment**

　　为哈希表 `key` 中的域 `field` 加上浮点数增量 `increment` 。

　　如果哈希表中没有域 `field` ，那么 `HINCRBYFLOAT` 会先将域 `field` 的值设为 `0` ，然后再执行加法操作。

　　如果键 `key` 不存在，那么 `HINCRBYFLOAT` 会先创建一个哈希表，再创建域 `field` ，最后再执行加法操作。

　　当以下任意一个条件发生时，返回一个错误：

- 域 `field` 的值不是字符串类型(因为 redis 中的数字和浮点数都以字符串的形式保存，所以它们都属于字符串类型）
- 域 `field` 当前的值或给定的增量 `increment` 不能解释(parse)为双精度浮点数(double precision floating point number)

　　`HINCRBYFLOAT` 命令的详细功能和 `INCRBYFLOAT` 命令类似，请查看 `INCRBYFLOAT` 命令获取更多相关信息。

　　**可用版本：** >= 2.6.0

　　**时间复杂度：**
O(1)

　　**返回值：**
执行加法操作之后 `field` 域的值。

　　::

```
# 值和增量都是普通小数

redis> HSET mykey field 10.50
(integer) 1
redis> HINCRBYFLOAT mykey field 0.1
"10.6"


# 值和增量都是指数符号

redis> HSET mykey field 5.0e3
(integer) 0
redis> HINCRBYFLOAT mykey field 2.0e2
"5200"


# 对不存在的键执行 HINCRBYFLOAT

redis> EXISTS price
(integer) 0
redis> HINCRBYFLOAT price milk 3.5
"3.5"
redis> HGETALL price
1) "milk"
2) "3.5"


# 对不存在的域进行 HINCRBYFLOAT

redis> HGETALL price
1) "milk"
2) "3.5"
redis> HINCRBYFLOAT price coffee 4.5   # 新增 coffee 域
"4.5"
redis> HGETALL price
1) "milk"
2) "3.5"
3) "coffee"
4) "4.5"
```

### HKEYS

　　**HKEYS key**

　　返回哈希表 `key` 中的所有域。

　　**可用版本：** >= 2.0.0

　　**时间复杂度：**
O(N)， `N` 为哈希表的大小。

　　**返回值：**
| 一个包含哈希表中所有域的表。
| 当 `key` 不存在时，返回一个空表。

　　::

```
# 哈希表非空

redis> HMSET website google www.google.com yahoo www.yahoo.com
OK

redis> HKEYS website
1) "google"
2) "yahoo"


# 空哈希表/key不存在

redis> EXISTS fake_key
(integer) 0

redis> HKEYS fake_key
(empty list or set)
```

### HLEN

　　**HLEN key**

　　返回哈希表 `key` 中域的数量。

　　**时间复杂度：**
O(1)

　　**返回值：**
| 哈希表中域的数量。
| 当 `key` 不存在时，返回 `0` 。

```
redis> HSET db redis redis.com
(integer) 1

redis> HSET db mysql mysql.com
(integer) 1

redis> HLEN db
(integer) 2

redis> HSET db mongodb mongodb.org
(integer) 1

redis> HLEN db
(integer) 3
```

### HMGET

　　**HMGET key field [field ...]**

　　返回哈希表 `key` 中，一个或多个给定域的值。

　　如果给定的域不存在于哈希表，那么返回一个 `nil` 值。

　　因为不存在的 `key` 被当作一个空哈希表来处理，所以对一个不存在的 `key` 进行 `HMGET` 操作将返回一个只带有 `nil` 值的表。

　　**可用版本：** >= 2.0.0

　　**时间复杂度：**
O(N)， `N` 为给定域的数量。

　　**返回值：**
一个包含多个给定域的关联值的表，表值的排列顺序和给定域参数的请求顺序一样。

```
redis> HMSET pet dog "doudou" cat "nounou"    # 一次设置多个域
OK

redis> HMGET pet dog cat fake_pet             # 返回值的顺序和传入参数的顺序一样
1) "doudou"
2) "nounou"
3) (nil)                                      # 不存在的域返回nil值
```

### HMSET

　　**HMSET key field value [field value ...]**

　　同时将多个 `field-value` (域-值)对设置到哈希表 `key` 中。

　　此命令会覆盖哈希表中已存在的域。

　　如果 `key` 不存在，一个空哈希表被创建并执行 `HMSET` 操作。

　　**可用版本：** >= 2.0.0

　　**时间复杂度：**
O(N)， `N` 为 `field-value` 对的数量。

　　**返回值：**
| 如果命令执行成功，返回 `OK` 。
| 当 `key` 不是哈希表(hash)类型时，返回一个错误。

```
redis> HMSET website google www.google.com yahoo www.yahoo.com
OK

redis> HGET website google
"www.google.com"

redis> HGET website yahoo
"www.yahoo.com"
```

### HSCAN

　　**HSCAN key cursor [MATCH pattern][count count]**

　　具体信息请参考 `SCAN` 命令。

### HSET

　　**HSET key field value**

　　将哈希表 `key` 中的域 `field` 的值设为 `value` 。

　　如果 `key` 不存在，一个新的哈希表被创建并进行 `HSET` 操作。

　　如果域 `field` 已经存在于哈希表中，旧值将被覆盖。

　　**可用版本：** >= 2.0.0

　　**时间复杂度：**
O(1)

　　**返回值：**
| 如果 `field` 是哈希表中的一个新建域，并且值设置成功，返回 `1` 。
| 如果哈希表中域 `field` 已经存在且旧值已被新值覆盖，返回 `0` 。

```
redis> HSET website google "www.g.cn"       # 设置一个新域
(integer) 1

redis> HSET website google "www.google.com" # 覆盖一个旧域
(integer) 0
```

### HSETNX

　　**HSETNX key field value**

　　将哈希表 `key` 中的域 `field` 的值设置为 `value` ，当且仅当域 `field` 不存在。

　　若域 `field` 已经存在，该操作无效。

　　如果 `key` 不存在，一个新哈希表被创建并执行 `HSETNX` 命令。

　　**可用版本：** >= 2.0.0

　　**时间复杂度：**
O(1)

　　**返回值：**
| 设置成功，返回 `1` 。
| 如果给定域已经存在且没有操作被执行，返回 `0` 。

```
redis> HSETNX nosql key-value-store redis
(integer) 1

redis> HSETNX nosql key-value-store redis       # 操作无效，域 key-value-store 已存在
(integer) 0
```

### HVALS

　　**HVALS key**

　　返回哈希表 `key` 中所有域的值。

　　**可用版本：** >= 2.0.0

　　**时间复杂度：**
O(N)， `N` 为哈希表的大小。

　　**返回值：**
| 一个包含哈希表中所有值的表。
| 当 `key` 不存在时，返回一个空表。

```
# 非空哈希表

redis> HMSET website google www.google.com yahoo www.yahoo.com
OK

redis> HVALS website
1) "www.google.com"
2) "www.yahoo.com"


# 空哈希表/不存在的key

redis> EXISTS not_exists
(integer) 0

redis> HVALS not_exists
(empty list or set)
```

## List（列表）

　　底层数据结构：可以是ziplist（压缩列表）和linkedlist（双端链表）

- 同时满足下面两个条件时使用压缩列表：
  - 列表保存元素个数小于512个
  - 每个元素长度小于64字节
- 不能满足上面两个条件使用linkedlist（双端列表）编码

### LINDEX

　　**LINDEX key index**

　　返回列表 `key` 中，下标为 `index` 的元素。

　　下标(index)参数 `start` 和 `stop` 都以 `0` 为底，也就是说，以 `0` 表示列表的第一个元素，以 `1` 表示列表的第二个元素，以此类推。

　　你也可以使用负数下标，以 `-1` 表示列表的最后一个元素， `-2` 表示列表的倒数第二个元素，以此类推。

　　如果 `key` 不是列表类型，返回一个错误。

　　**可用版本：**

> = 1.0.0

　　**时间复杂度：**
| O(N)， `N` 为到达下标 `index` 过程中经过的元素数量。
| 因此，对列表的头元素和尾元素执行 `LINDEX`_ 命令，复杂度为O(1)。

　　**返回值:**
| 列表中下标为 `index` 的元素。
| 如果 `index` 参数的值不在列表的区间范围内(out of range)，返回 `nil` 。

　　::

```
redis> LPUSH mylist "World"
(integer) 1

redis> LPUSH mylist "Hello"
(integer) 2

redis> LINDEX mylist 0
"Hello"

redis> LINDEX mylist -1
"World"

redis> LINDEX mylist 3        # index不在 mylist 的区间范围内
(nil)
```

### LINSERT

　　**LINSERT key BEFORE|AFTER pivot value**

　　将值 `value` 插入到列表 `key` 当中，位于值 `pivot` 之前或之后。

　　当 `pivot` 不存在于列表 `key` 时，不执行任何操作。

　　当 `key` 不存在时， `key` 被视为空列表，不执行任何操作。

　　如果 `key` 不是列表类型，返回一个错误。

　　**可用版本：**

> = 2.2.0

　　**时间复杂度:**
O(N)， `N` 为寻找 `pivot` 过程中经过的元素数量。

　　**返回值:**
| 如果命令执行成功，返回插入操作完成之后，列表的长度。
| 如果没有找到 `pivot` ，返回 `-1` 。
| 如果 `key` 不存在或为空列表，返回 `0` 。

　　::

```
redis> RPUSH mylist "Hello"
(integer) 1

redis> RPUSH mylist "World"
(integer) 2

redis> LINSERT mylist BEFORE "World" "There"
(integer) 3

redis> LRANGE mylist 0 -1
1) "Hello"
2) "There"
3) "World"


# 对一个非空列表插入，查找一个不存在的 pivot

redis> LINSERT mylist BEFORE "go" "let's"    
(integer) -1                                    # 失败


# 对一个空列表执行 LINSERT 命令

redis> EXISTS fake_list  
(integer) 0

redis> LINSERT fake_list BEFORE "nono" "gogogog"
(integer) 0                                      # 失败
```

### LLEN

　　**LLEN key**

　　返回列表 `key` 的长度。

　　如果 `key` 不存在，则 `key` 被解释为一个空列表，返回 `0` .

　　如果 `key` 不是列表类型，返回一个错误。

　　**可用版本：**

> = 1.0.0

　　**时间复杂度：**
O(1)

　　**返回值：**
列表 `key` 的长度。

　　::

```
# 空列表

redis> LLEN job 
(integer) 0


# 非空列表

redis> LPUSH job "cook food"
(integer) 1

redis> LPUSH job "have lunch"
(integer) 2

redis> LLEN job
(integer) 2
```

### LPOP

　　**LPOP key**

　　移除并返回列表 `key` 的头元素。

　　**可用版本：**

> = 1.0.0

　　**时间复杂度：**
O(1)

　　**返回值：**
| 列表的头元素。
| 当 `key` 不存在时，返回 `nil` 。

　　::

```
redis> LLEN course
(integer) 0

redis> RPUSH course algorithm001
(integer) 1

redis> RPUSH course c++101
(integer) 2

redis> LPOP course  # 移除头元素
"algorithm001"
```

### LPUSH

　　**LPUSH key value [value ...]**

　　将一个或多个值 `value` 插入到列表 `key` 的表头

　　如果有多个 `value` 值，那么各个 `value` 值按从左到右的顺序依次插入到表头：
比如说，对空列表 `mylist` 执行命令 `LPUSH mylist a b c` ，列表的值将是 `c b a` ，这等同于原子性地执行 `LPUSH mylist a` 、 `LPUSH mylist b` 和 `LPUSH mylist c` 三个命令。

　　如果 `key` 不存在，一个空列表会被创建并执行 `LPUSH`_ 操作。

　　当 `key` 存在但不是列表类型时，返回一个错误。

> 在Redis 2.4版本以前的 `LPUSH`_ 命令，都只接受单个 `value` 值。

　　**可用版本：**

> = 1.0.0

　　**时间复杂度：**
O(1)

　　**返回值：**
执行 `LPUSH`_ 命令后，列表的长度。

# 加入单个元素

```
redis> LPUSH languages python
(integer) 1


# 加入重复元素

redis> LPUSH languages python
(integer) 2

redis> LRANGE languages 0 -1     # 列表允许重复元素
1) "python"
2) "python"


# 加入多个元素

redis> LPUSH mylist a b c
(integer) 3

redis> LRANGE mylist 0 -1
1) "c"
2) "b"
3) "a"
```

### LPUSHX

　　**LPUSHX key value**

　　将值 `value` 插入到列表 `key` 的表头，当且仅当 `key` 存在并且是一个列表。

　　和 `LPUSH` 命令相反，当 `key` 不存在时， `LPUSHX`_ 命令什么也不做。

　　**可用版本：**

> = 2.2.0

　　**时间复杂度：**
O(1)

　　**返回值：**
`LPUSHX`_ 命令执行之后，表的长度。

　　::

```
# 对空列表执行 LPUSHX

redis> LLEN greet                       # greet 是一个空列表
(integer) 0

redis> LPUSHX greet "hello"             # 尝试 LPUSHX，失败，因为列表为空
(integer) 0
```

# 对非空列表执行 LPUSHX

```
redis> LPUSH greet "hello"              # 先用 LPUSH 创建一个有一个元素的列表
(integer) 1

redis> LPUSHX greet "good morning"      # 这次 LPUSHX 执行成功
(integer) 2

redis> LRANGE greet 0 -1
1) "good morning"
2) "hello"
```

### LRANGE

　　**LRANGE key start stop**

　　返回列表 `key` 中指定区间内的元素，区间以偏移量 `start` 和 `stop` 指定。

　　下标(index)参数 `start` 和 `stop` 都以 `0` 为底，也就是说，以 `0` 表示列表的第一个元素，以 `1` 表示列表的第二个元素，以此类推。

　　你也可以使用负数下标，以 `-1` 表示列表的最后一个元素， `-2` 表示列表的倒数第二个元素，以此类推。

　　**注意LRANGE命令和编程语言区间函数的区别**

　　假如你有一个包含一百个元素的列表，对该列表执行 `LRANGE list 0 10` ，结果是一个包含11个元素的列表，这表明 `stop` 下标也在 `LRANGE`_ 命令的取值范围之内(闭区间)，这和某些语言的区间函数可能不一致，比如Ruby的 `Range.new` 、 `Array#slice` 和Python的 `range()` 函数。

　　**超出范围的下标**

　　超出范围的下标值不会引起错误。

　　如果 `start` 下标比列表的最大下标 `end` ( `LLEN list` 减去 `1` )还要大，那么 `LRANGE`_ 返回一个空列表。

　　如果 `stop` 下标比 `end` 下标还要大，Redis将 `stop` 的值设置为 `end` 。

　　**可用版本：**

> = 1.0.0

　　**时间复杂度:**
O(S+N)， `S` 为偏移量 `start` ， `N` 为指定区间内元素的数量。

　　**返回值:**
一个列表，包含指定区间内的元素。

　　::

```
redis> RPUSH fp-language lisp   
(integer) 1

redis> LRANGE fp-language 0 0 
1) "lisp"

redis> RPUSH fp-language scheme
(integer) 2

redis> LRANGE fp-language 0 1
1) "lisp"
2) "scheme"
```

### LREM

　　**LREM key count value**

　　根据参数 `count` 的值，移除列表中与参数 `value` 相等的元素。

　　`count` 的值可以是以下几种：

- `count > 0` : 从表头开始向表尾搜索，移除与 `value` 相等的元素，数量为 `count` 。
- `count < 0` : 从表尾开始向表头搜索，移除与 `value` 相等的元素，数量为 `count` 的绝对值。
- `count = 0` : 移除表中所有与 `value` 相等的值。

　　**可用版本：**

> = 1.0.0

　　**时间复杂度：**
O(N)， `N` 为列表的长度。

　　**返回值：**
| 被移除元素的数量。
| 因为不存在的 `key` 被视作空表(empty list)，所以当 `key` 不存在时， `LREM`_ 命令总是返回 `0` 。

　　::

```
# 先创建一个表，内容排列是
# morning hello morning helllo morning

redis> LPUSH greet "morning"
(integer) 1
redis> LPUSH greet "hello"
(integer) 2
redis> LPUSH greet "morning"
(integer) 3
redis> LPUSH greet "hello"
(integer) 4
redis> LPUSH greet "morning"
(integer) 5

redis> LRANGE greet 0 4         # 查看所有元素
1) "morning"
2) "hello"
3) "morning"
4) "hello"
5) "morning"

redis> LREM greet 2 morning     # 移除从表头到表尾，最先发现的两个 morning
(integer) 2                     # 两个元素被移除

redis> LLEN greet               # 还剩 3 个元素
(integer) 3

redis> LRANGE greet 0 2
1) "hello"
2) "hello"
3) "morning"

redis> LREM greet -1 morning    # 移除从表尾到表头，第一个 morning
(integer) 1

redis> LLEN greet               # 剩下两个元素
(integer) 2

redis> LRANGE greet 0 1
1) "hello"
2) "hello"

redis> LREM greet 0 hello      # 移除表中所有 hello
(integer) 2                    # 两个 hello 被移除

redis> LLEN greet
(integer) 0
```

### LSET

　　**LSET key index value**

　　将列表 `key` 下标为 `index` 的元素的值设置为 `value` 。

　　当 `index` 参数超出范围，或对一个空列表( `key` 不存在)进行 `LSET`_ 时，返回一个错误。

　　关于列表下标的更多信息，请参考 `LINDEX` 命令。

　　**可用版本：**

> = 1.0.0

　　**时间复杂度：**
| 对头元素或尾元素进行 `LSET`_ 操作，复杂度为 O(1)。
| 其他情况下，为 O(N)， `N` 为列表的长度。

　　**返回值：**
操作成功返回 `ok` ，否则返回错误信息。

　　::

```
# 对空列表(key 不存在)进行 LSET

redis> EXISTS list
(integer) 0

redis> LSET list 0 item
(error) ERR no such key


# 对非空列表进行 LSET

redis> LPUSH job "cook food"
(integer) 1

redis> LRANGE job 0 0
1) "cook food"

redis> LSET job 0 "play game"
OK

redis> LRANGE job  0 0
1) "play game"


# index 超出范围

redis> LLEN list                    # 列表长度为 1
(integer) 1

redis> LSET list 3 'out of range'
(error) ERR index out of range
```

### LTRIM

　　**LTRIM key start stop**

　　对一个列表进行修剪(trim)，就是说，让列表只保留指定区间内的元素，不在指定区间之内的元素都将被删除。

　　举个例子，执行命令 `LTRIM list 0 2` ，表示只保留列表 `list` 的前三个元素，其余元素全部删除。

　　下标(index)参数 `start` 和 `stop` 都以 `0` 为底，也就是说，以 `0` 表示列表的第一个元素，以 `1` 表示列表的第二个元素，以此类推。

　　你也可以使用负数下标，以 `-1` 表示列表的最后一个元素， `-2` 表示列表的倒数第二个元素，以此类推。

　　当 `key` 不是列表类型时，返回一个错误。

　　`LTRIM`_ 命令通常和 `LPUSH` 命令或 `RPUSH` 命令配合使用，举个例子：

　　::

```
LPUSH log newest_log
LTRIM log 0 99
```

　　这个例子模拟了一个日志程序，每次将最新日志 `newest_log` 放到 `log` 列表中，并且只保留最新的 `100` 项。注意当这样使用 `LTRIM` 命令时，时间复杂度是O(1)，因为平均情况下，每次只有一个元素被移除。

　　**注意LTRIM命令和编程语言区间函数的区别**

　　假如你有一个包含一百个元素的列表 `list` ，对该列表执行 `LTRIM list 0 10` ，结果是一个包含11个元素的列表，这表明 `stop` 下标也在 `LTRIM`_ 命令的取值范围之内(闭区间)，这和某些语言的区间函数可能不一致，比如Ruby的 `Range.new` 、 `Array#slice` 和Python的 `range()` 函数。

　　**超出范围的下标**

　　超出范围的下标值不会引起错误。

　　如果 `start` 下标比列表的最大下标 `end` ( `LLEN list` 减去 `1` )还要大，或者 `start > stop` ， `LTRIM`_ 返回一个空列表(因为 _`_LTRIM_` 已经将整个列表清空)。

　　如果 `stop` 下标比 `end` 下标还要大，Redis将 `stop` 的值设置为 `end` 。

　　**可用版本：**

> = 1.0.0

　　**时间复杂度:**
O(N)， `N` 为被移除的元素的数量。

　　**返回值:**
| 命令执行成功时，返回 `ok` 。

　　::

```
# 情况 1： 常见情况， start 和 stop 都在列表的索引范围之内

redis> LRANGE alpha 0 -1       # alpha 是一个包含 5 个字符串的列表
1) "h"
2) "e"
3) "l"
4) "l"
5) "o"

redis> LTRIM alpha 1 -1        # 删除 alpha 列表索引为 0 的元素
OK

redis> LRANGE alpha 0 -1       # "h" 被删除了
1) "e"
2) "l"
3) "l"
4) "o"


# 情况 2： stop 比列表的最大下标还要大


redis> LTRIM alpha 1 10086     # 保留 alpha 列表索引 1 至索引 10086 上的元素
OK

redis> LRANGE alpha 0 -1       # 只有索引 0 上的元素 "e" 被删除了，其他元素还在
1) "l"
2) "l"
3) "o"
```

# 情况 3： start 和 stop 都比列表的最大下标要大，并且 start < stop

```
redis> LTRIM alpha 10086 123321
OK

redis> LRANGE alpha 0 -1        # 列表被清空
(empty list or set)



# 情况 4： start 和 stop 都比列表的最大下标要大，并且 start > stop

redis> RPUSH new-alpha "h" "e" "l" "l" "o"     # 重新建立一个新列表
(integer) 5

redis> LRANGE new-alpha 0 -1
1) "h"
2) "e"
3) "l"
4) "l"
5) "o"

redis> LTRIM new-alpha 123321 10086    # 执行 LTRIM
OK

redis> LRANGE new-alpha 0 -1           # 同样被清空
(empty list or set)
```

### RPOP

　　**RPOP key**

　　移除并返回列表 `key` 的尾元素。

　　**可用版本：**

> = 1.0.0

　　**时间复杂度：**
O(1)

　　**返回值：**
| 列表的尾元素。
| 当 `key` 不存在时，返回 `nil` 。

　　::

```
redis> RPUSH mylist "one" 
(integer) 1

redis> RPUSH mylist "two"
(integer) 2

redis> RPUSH mylist "three"
(integer) 3

redis> RPOP mylist           # 返回被弹出的元素
"three"

redis> LRANGE mylist 0 -1    # 列表剩下的元素 
1) "one"
2) "two"
```

### RPOPLPUSH

　　**RPOPLPUSH source destination**

　　命令 `RPOPLPUSH`_ 在一个原子时间内，执行以下两个动作：

- 将列表 `source` 中的最后一个元素(尾元素)弹出，并返回给客户端。
- 将 `source` 弹出的元素插入到列表 `destination` ，作为 `destination` 列表的的头元素。

　　举个例子，你有两个列表 `source` 和 `destination` ， `source` 列表有元素 `a, b, c` ， `destination` 列表有元素 `x, y, z` ，执行 `RPOPLPUSH source destination` 之后， `source` 列表包含元素 `a, b` ， `destination` 列表包含元素 `c, x, y, z`  ，并且元素 `c` 会被返回给客户端。

　　如果 `source` 不存在，值 `nil` 被返回，并且不执行其他动作。

　　如果 `source` 和 `destination` 相同，则列表中的表尾元素被移动到表头，并返回该元素，可以把这种特殊情况视作列表的旋转(rotation)操作。

　　**可用版本：**

> = 1.2.0

　　**时间复杂度：**
O(1)

　　**返回值：**
被弹出的元素。

　　::

```
# source 和 destination 不同

redis> LRANGE alpha 0 -1         # 查看所有元素
1) "a"
2) "b"
3) "c"
4) "d"

redis> RPOPLPUSH alpha reciver   # 执行一次 RPOPLPUSH 看看
"d"

redis> LRANGE alpha 0 -1 
1) "a"
2) "b"
3) "c"

redis> LRANGE reciver 0 -1
1) "d"

redis> RPOPLPUSH alpha reciver   # 再执行一次，证实 RPOP 和 LPUSH 的位置正确
"c"

redis> LRANGE alpha 0 -1
1) "a"
2) "b"

redis> LRANGE reciver 0 -1
1) "c"
2) "d"
```

# source 和 destination 相同

```
redis> LRANGE number 0 -1
1) "1"
2) "2"
3) "3"
4) "4"

redis> RPOPLPUSH number number
"4"

redis> LRANGE number 0 -1           # 4 被旋转到了表头
1) "4"
2) "1"
3) "2"
4) "3"

redis> RPOPLPUSH number number
"3"

redis> LRANGE number 0 -1           # 这次是 3 被旋转到了表头
1) "3"
2) "4"
3) "1"
4) "2"
```

#### 模式： 安全的队列

---

　　Redis的列表经常被用作队列(queue)，用于在不同程序之间有序地交换消息(message)。一个客户端通过 `LPUSH` 命令将消息放入队列中，而另一个客户端通过 `RPOP` 或者 `BRPOP` 命令取出队列中等待时间最长的消息。

　　不幸的是，上面的队列方法是『不安全』的，因为在这个过程中，一个客户端可能在取出一个消息之后崩溃，而未处理完的消息也就因此丢失。

　　使用 `RPOPLPUSH`_ 命令(或者它的阻塞版本 `BRPOPLPUSH` )可以解决这个问题：因为它不仅返回一个消息，同时还将这个消息添加到另一个备份列表当中，如果一切正常的话，当一个客户端完成某个消息的处理之后，可以用 `LREM` 命令将这个消息从备份表删除。

　　最后，还可以添加一个客户端专门用于监视备份表，它自动地将超过一定处理时限的消息重新放入队列中去(负责处理该消息的客户端可能已经崩溃)，这样就不会丢失任何消息了。

#### 模式：循环列表

---

　　通过使用相同的 `key` 作为 `RPOPLPUSH`_ 命令的两个参数，客户端可以用一个接一个地获取列表元素的方式，取得列表的所有元素，而不必像 `LRANGE` 命令那样一下子将所有列表元素都从服务器传送到客户端中(两种方式的总复杂度都是 O(N))。

　　以上的模式甚至在以下的两个情况下也能正常工作：

- 有多个客户端同时对同一个列表进行旋转(rotating)，它们获取不同的元素，直到所有元素都被读取完，之后又从头开始。
- 有客户端在向列表尾部(右边)添加新元素。

　　这个模式使得我们可以很容易实现这样一类系统：有 N 个客户端，需要连续不断地对一些元素进行处理，而且处理的过程必须尽可能地快。一个典型的例子就是服务器的监控程序：它们需要在尽可能短的时间内，并行地检查一组网站，确保它们的可访问性。

　　注意，使用这个模式的客户端是易于扩展(scala)且安全(reliable)的，因为就算接收到元素的客户端失败，元素还是保存在列表里面，不会丢失，等到下个迭代来临的时候，别的客户端又可以继续处理这些元素了。

### RPUSH

　　**RPUSH key value [value ...]**

　　将一个或多个值 `value` 插入到列表 `key` 的表尾(最右边)。

　　如果有多个 `value` 值，那么各个 `value` 值按从左到右的顺序依次插入到表尾：比如对一个空列表 `mylist` 执行 `RPUSH mylist a b c` ，得出的结果列表为 `a b c` ，等同于执行命令 `RPUSH mylist a` 、 `RPUSH mylist b` 、 `RPUSH mylist c` 。

　　如果 `key` 不存在，一个空列表会被创建并执行 `RPUSH`_ 操作。

　　当 `key` 存在但不是列表类型时，返回一个错误。

> 在 Redis 2.4 版本以前的 `RPUSH`_ 命令，都只接受单个 `value` 值。

　　**可用版本：**

> = 1.0.0

　　**时间复杂度：**
O(1)

　　**返回值：**
执行 `RPUSH`_ 操作后，表的长度。

　　::

```
# 添加单个元素

redis> RPUSH languages c
(integer) 1


# 添加重复元素

redis> RPUSH languages c
(integer) 2

redis> LRANGE languages 0 -1 # 列表允许重复元素
1) "c"
2) "c"


# 添加多个元素

redis> RPUSH mylist a b c
(integer) 3

redis> LRANGE mylist 0 -1
1) "a"
2) "b"
3) "c"
```

### RPUSHX

　　**RPUSHX key value**

　　将值 `value` 插入到列表 `key` 的表尾，当且仅当 `key` 存在并且是一个列表。

　　和 `RPUSH` 命令相反，当 `key` 不存在时， `RPUSHX`_ 命令什么也不做。

　　**可用版本：**

> = 2.2.0

　　**时间复杂度：**
O(1)

　　**返回值：**
`RPUSHX`_ 命令执行之后，表的长度。

　　::

```
# key不存在

redis> LLEN greet
(integer) 0

redis> RPUSHX greet "hello"     # 对不存在的 key 进行 RPUSHX，PUSH 失败。
(integer) 0
```

# key 存在且是一个非空列表

```
redis> RPUSH greet "hi"         # 先用 RPUSH 插入一个元素
(integer) 1

redis> RPUSHX greet "hello"     # greet 现在是一个列表类型，RPUSHX 操作成功。
(integer) 2

redis> LRANGE greet 0 -1
1) "hi"
2) "hello"
```

### BLPOP

　　**BLPOP key [key ...] timeout**

　　`BLPOP`_ 是列表的阻塞式(blocking)弹出原语。

　　它是 `LPOP` 命令的阻塞版本，当给定列表内没有任何元素可供弹出的时候，连接将被 `BLPOP`_ 命令阻塞，直到等待超时或发现可弹出元素为止。

　　当给定多个 `key` 参数时，按参数 `key` 的先后顺序依次检查各个列表，弹出第一个非空列表的头元素。

　　**非阻塞行为**

　　当 `BLPOP`_ 被调用时，如果给定 `key` 内至少有一个非空列表，那么弹出遇到的第一个非空列表的头元素，并和被弹出元素所属的列表的名字一起，组成结果返回给调用者。

　　当存在多个给定 `key` 时， `BLPOP`_ 按给定 `key` 参数排列的先后顺序，依次检查各个列表。

　　假设现在有 `job` 、  `command` 和 `request` 三个列表，其中 `job` 不存在， `command` 和 `request` 都持有非空列表。考虑以下命令：

　　`BLPOP job command request 0`

　　`BLPOP`_ 保证返回的元素来自 `command` ，因为它是按"查找 `job`  -> 查找 `command`  -> 查找 `request` "这样的顺序，第一个找到的非空列表。

　　::

```
redis> DEL job command request           # 确保key都被删除
(integer) 0

redis> LPUSH command "update system..."  # 为command列表增加一个值
(integer) 1

redis> LPUSH request "visit page"        # 为request列表增加一个值
(integer) 1

redis> BLPOP job command request 0       # job 列表为空，被跳过，紧接着 command 列表的第一个元素被弹出。
1) "command"                             # 弹出元素所属的列表
2) "update system..."                    # 弹出元素所属的值
```

　　**阻塞行为**

　　如果所有给定 `key` 都不存在或包含空列表，那么 `BLPOP`_ 命令将阻塞连接，直到等待超时，或有另一个客户端对给定 `key` 的任意一个执行 `LPUSH` 或 `RPUSH` 命令为止。

　　超时参数 `timeout` 接受一个以秒为单位的数字作为值。超时参数设为 `0` 表示阻塞时间可以无限期延长(block indefinitely) 。

　　::

```
redis> EXISTS job                # 确保两个 key 都不存在
(integer) 0
redis> EXISTS command
(integer) 0

redis> BLPOP job command 300     # 因为key一开始不存在，所以操作会被阻塞，直到另一客户端对 job 或者 command 列表进行 PUSH 操作。
1) "job"                         # 这里被 push 的是 job
2) "do my home work"             # 被弹出的值
(26.26s)                         # 等待的秒数

redis> BLPOP job command 5       # 等待超时的情况
(nil)
(5.66s)                          # 等待的秒数
```

　　**相同的key被多个客户端同时阻塞**

　　相同的 `key` 可以被多个客户端同时阻塞。

　　不同的客户端被放进一个队列中，按『先阻塞先服务』(first-BLPOP，first-served)的顺序为 `key` 执行 `BLPOP`_ 命令。

　　**在MULTI/EXEC事务中的BLPOP**

　　`BLPOP`_ 可以用于流水线(pipline,批量地发送多个命令并读入多个回复)，但把它用在 `multi` / `exec` 块当中没有意义。因为这要求整个服务器被阻塞以保证块执行时的原子性，该行为阻止了其他客户端执行 `LPUSH` 或 `RPUSH` 命令。

　　因此，一个被包裹在 `multi` / `exec` 块内的 `BLPOP`_ 命令，行为表现得就像 `LPOP` 一样，对空列表返回 `nil` ，对非空列表弹出列表元素，不进行任何阻塞操作。

　　::

```
# 对非空列表进行操作

redis> RPUSH job programming
(integer) 1

redis> MULTI
OK

redis> BLPOP job 30
QUEUED

redis> EXEC           # 不阻塞，立即返回
1) 1) "job"
   2) "programming"


# 对空列表进行操作

redis> LLEN job      # 空列表
(integer) 0

redis> MULTI
OK

redis> BLPOP job 30
QUEUED

redis> EXEC         # 不阻塞，立即返回
1) (nil)
```

　　**可用版本：**

> = 2.0.0

　　**时间复杂度：**
O(1)

　　**返回值：**
| 如果列表为空，返回一个 `nil` 。
| 否则，返回一个含有两个元素的列表，第一个元素是被弹出元素所属的 `key` ，第二个元素是被弹出元素的值。

#### 模式：事件提醒

　　有时候，为了等待一个新元素到达数据中，需要使用轮询的方式对数据进行探查。

　　另一种更好的方式是，使用系统提供的阻塞原语，在新元素到达时立即进行处理，而新元素还没到达时，就一直阻塞住，避免轮询占用资源。

　　对于 Redis ，我们似乎需要一个阻塞版的 `SPOP` 命令，但实际上，使用 `BLPOP`_ 或者 `BRPOP` 就能很好地解决这个问题。

　　使用元素的客户端(消费者)可以执行类似以下的代码：

```
LOOP forever
    WHILE SPOP(key) returns elements
        ... process elements ...
    END
    BRPOP helper_key
END
```

　　添加元素的客户端(生产者)则执行以下代码：

```
MULTI
    SADD key element
    LPUSH helper_key x
EXEC
```

### BRPOP

　　**BRPOP key [key ...] timeout**

　　`BRPOP`_ 是列表的阻塞式(blocking)弹出原语。

　　它是 `RPOP` 命令的阻塞版本，当给定列表内没有任何元素可供弹出的时候，连接将被 `BRPOP`_ 命令阻塞，直到等待超时或发现可弹出元素为止。

　　当给定多个 `key` 参数时，按参数 `key` 的先后顺序依次检查各个列表，弹出第一个非空列表的尾部元素。

　　关于阻塞操作的更多信息，请查看 `BLPOP` 命令， `BRPOP`_ 除了弹出元素的位置和 `BLPOP` 不同之外，其他表现一致。

　　**可用版本：**

> = 2.0.0

　　**时间复杂度：**
O(1)

　　**返回值：**
| 假如在指定时间内没有任何元素被弹出，则返回一个 `nil` 和等待时长。
| 反之，返回一个含有两个元素的列表，第一个元素是被弹出元素所属的 `key` ，第二个元素是被弹出元素的值。

　　::

```
redis> LLEN course
(integer) 0

redis> RPUSH course algorithm001
(integer) 1

redis> RPUSH course c++101
(integer) 2

redis> BRPOP course 30
1) "course"             # 被弹出元素所属的列表键
2) "c++101"             # 被弹出的元素
```

### BRPOPLPUSH

　　**BRPOPLPUSH source destination timeout**

　　`BRPOPLPUSH`_ 是 *`_RPOPLPUSH_`* 的阻塞版本，当给定列表 *`_source_`* 不为空时， _`_BRPOPLPUSH_` 的表现和 `RPOPLPUSH` 一样。

　　当列表 `source` 为空时， `BRPOPLPUSH`_ 命令将阻塞连接，直到等待超时，或有另一个客户端对 `source` 执行 `LPUSH` 或 `RPUSH` 命令为止。

　　超时参数 `timeout` 接受一个以秒为单位的数字作为值。超时参数设为 `0` 表示阻塞时间可以无限期延长(block indefinitely) 。

　　更多相关信息，请参考 `RPOPLPUSH` 命令。

　　**可用版本：**

> = 2.2.0

　　**时间复杂度：**
O(1)

　　**返回值：**
| 假如在指定时间内没有任何元素被弹出，则返回一个 `nil` 和等待时长。
| 反之，返回一个含有两个元素的列表，第一个元素是被弹出元素的值，第二个元素是等待时长。

　　::

```
# 非空列表

redis> BRPOPLPUSH msg reciver 500
"hello moto"                        # 弹出元素的值
(3.38s)                             # 等待时长

redis> LLEN reciver
(integer) 1

redis> LRANGE reciver 0 0
1) "hello moto"


# 空列表

redis> BRPOPLPUSH msg reciver 1 
(nil)
(1.34s)
```

#### 模式：安全队列

　　参考 `RPOPLPUSH` 命令的『安全队列』模式。

#### 模式：循环列表

　　参考 `RPOPLPUSH` 命令的『循环列表』模式。

## Set

　　**集合对象set是string类型（整数也会转成string类型进行存储）的无序集合。注意集合和列表的区别：集合中的元素是无序的，因此不能通过索引来操作元素；集合中的元素不能有重复。**
**底层数据结构：**可以是intset或者hashtable

- intset编码的集合对象使用整数集合作为底层实现，集合对象包含的所有元素都被保存在整数集合中。
- hashtable编码的集合对象使用字典作为底层实现，字典的每个键都是一个字符串对象，这里的每个字符串对象就是一个集合中的元素，而字典的值全部设置为null。**当使用HT编码时，Redis中的集合SET相当于Java中的HashSet，内部的键值对是无序的，唯一的。内部实现相当于一个特殊的字典，字典中所有value都是NULL**

　　当集合满足下列两个条件时，使用intset编码：

- 集合对象中的所有元素都是整数
- 集合对象所有元素数量不超过512

### SADD

　　**SADD key member [member ...]**

　　将一个或多个 `member` 元素加入到集合 `key` 当中，已经存在于集合的 `member` 元素将被忽略。

　　假如 `key` 不存在，则创建一个只包含 `member` 元素作成员的集合。

　　当 `key` 不是集合类型时，返回一个错误。

> 在Redis2.4版本以前， `SADD`_ 只接受单个 `member` 值。

　　**可用版本：**

> = 1.0.0

　　**时间复杂度:**
O(N)， `N` 是被添加的元素的数量。

　　**返回值:**
被添加到集合中的新元素的数量，不包括被忽略的元素。

　　::

```
# 添加单个元素

redis> SADD bbs "discuz.net"
(integer) 1


# 添加重复元素 

redis> SADD bbs "discuz.net"
(integer) 0


# 添加多个元素

redis> SADD bbs "tianya.cn" "groups.google.com"
(integer) 2

redis> SMEMBERS bbs
1) "discuz.net"
2) "groups.google.com"
3) "tianya.cn"
```

### SCARD

　　**SCARD key**

　　返回集合 `key` 的基数(集合中元素的数量)。

　　**可用版本：**

> = 1.0.0

　　**时间复杂度:**
O(1)

　　**返回值：**
| 集合的基数。
| 当 `key` 不存在时，返回 `0` 。

　　::

```
redis> SADD tool pc printer phone
(integer) 3

redis> SCARD tool   # 非空集合
(integer) 3

redis> DEL tool
(integer) 1

redis> SCARD tool   # 空集合
(integer) 0
```

### SDIFF

　　**SDIFF key [key ...]**

　　返回一个集合的全部成员，该集合是所有给定集合之间的差集。

　　不存在的 `key` 被视为空集。

　　**可用版本：**

> = 1.0.0

　　**时间复杂度:**
O(N)， `N` 是所有给定集合的成员数量之和。

　　**返回值:**
一个包含差集成员的列表。

　　::

```
redis> SMEMBERS peter's_movies
1) "bet man"
2) "start war"
3) "2012"

redis> SMEMBERS joe's_movies
1) "hi, lady"
2) "Fast Five"
3) "2012"

redis> SDIFF peter's_movies joe's_movies
1) "bet man"
2) "start war"
```

### SDIFFSTORE

　　**SDIFFSTORE destination key [key ...]**

　　这个命令的作用和 `SDIFF` 类似，但它将结果保存到 `destination` 集合，而不是简单地返回结果集。

　　如果 `destination` 集合已经存在，则将其覆盖。

　　`destination` 可以是 `key` 本身。

　　**可用版本：**

> = 1.0.0

　　**时间复杂度:**
O(N)， `N` 是所有给定集合的成员数量之和。

　　**返回值:**
结果集中的元素数量。

　　::

```
redis> SMEMBERS joe's_movies
1) "hi, lady"
2) "Fast Five"
3) "2012"   

redis> SMEMBERS peter's_movies
1) "bet man"
2) "start war"
3) "2012"

redis> SDIFFSTORE joe_diff_peter joe's_movies peter's_movies
(integer) 2

redis> SMEMBERS joe_diff_peter
1) "hi, lady"
2) "Fast Five"
```

### SINTER

　　**SINTER key [key ...]**

　　返回一个集合的全部成员，该集合是所有给定集合的交集。

　　不存在的 `key` 被视为空集。

　　当给定集合当中有一个空集时，结果也为空集(根据集合运算定律)。

　　**可用版本：**

> = 1.0.0

　　**时间复杂度:**
O(N * M)， `N` 为给定集合当中基数最小的集合， `M` 为给定集合的个数。

　　**返回值:**
交集成员的列表。

　　::

```
redis> SMEMBERS group_1
1) "LI LEI"
2) "TOM"
3) "JACK"

redis> SMEMBERS group_2
1) "HAN MEIMEI"
2) "JACK" 

redis> SINTER group_1 group_2
1) "JACK"
```

### SINTERSTORE

　　**SINTERSTORE destination key [key ...]**

　　这个命令类似于 `SINTER` 命令，但它将结果保存到 `destination` 集合，而不是简单地返回结果集。

　　如果 `destination` 集合已经存在，则将其覆盖。

　　`destination` 可以是 `key` 本身。

　　**可用版本：**

> = 1.0.0

　　**时间复杂度:**
O(N * M)， `N` 为给定集合当中基数最小的集合， `M` 为给定集合的个数。

　　**返回值:**
结果集中的成员数量。

　　::

```
redis> SMEMBERS songs
1) "good bye joe"
2) "hello,peter"

redis> SMEMBERS my_songs
1) "good bye joe"
2) "falling"

redis> SINTERSTORE song_interset songs my_songs
(integer) 1

redis> SMEMBERS song_interset
1) "good bye joe"
```

### SISMEMBER

　　**SISMEMBER key member**

　　判断 `member` 元素是否集合 `key` 的成员。

　　**可用版本：**

> = 1.0.0

　　**时间复杂度:**
O(1)

　　**返回值:**
| 如果 `member` 元素是集合的成员，返回 `1` 。
| 如果 `member` 元素不是集合的成员，或 `key` 不存在，返回 `0` 。

　　::

```
redis> SMEMBERS joe's_movies
1) "hi, lady"
2) "Fast Five"
3) "2012"

redis> SISMEMBER joe's_movies "bet man"
(integer) 0

redis> SISMEMBER joe's_movies "Fast Five"
(integer) 1
```

### SMEMBERS

　　**SMEMBERS key**

　　返回集合 `key` 中的所有成员。

　　不存在的 `key` 被视为空集合。

　　**可用版本：**

> = 1.0.0

　　**时间复杂度:**
O(N)， `N` 为集合的基数。

　　**返回值:**
集合中的所有成员。

　　::

```
# key 不存在或集合为空

redis> EXISTS not_exists_key
(integer) 0

redis> SMEMBERS not_exists_key
(empty list or set)


# 非空集合

redis> SADD language Ruby Python Clojure
(integer) 3

redis> SMEMBERS language
1) "Python"
2) "Ruby"
3) "Clojure"
```

### SMOVE

　　**SMOVE source destination member**

　　将 `member` 元素从 `source` 集合移动到 `destination` 集合。

　　`SMOVE`_ 是原子性操作。

　　如果 `source` 集合不存在或不包含指定的 `member` 元素，则 `SMOVE`_ 命令不执行任何操作，仅返回 `0` 。否则， `member` 元素从 `source` 集合中被移除，并添加到 `destination` 集合中去。

　　当 `destination` 集合已经包含 `member` 元素时， `SMOVE`_ 命令只是简单地将 `source` 集合中的 `member` 元素删除。

　　当 `source` 或 `destination` 不是集合类型时，返回一个错误。

　　**可用版本：**

> = 1.0.0

　　**时间复杂度:**
O(1)

　　**返回值:**
| 如果 `member` 元素被成功移除，返回 `1` 。
| 如果 `member` 元素不是 `source` 集合的成员，并且没有任何操作对 `destination` 集合执行，那么返回 `0` 。

　　::

```
redis> SMEMBERS songs
1) "Billie Jean"
2) "Believe Me"

redis> SMEMBERS my_songs
(empty list or set)

redis> SMOVE songs my_songs "Believe Me"
(integer) 1

redis> SMEMBERS songs
1) "Billie Jean"

redis> SMEMBERS my_songs
1) "Believe Me"
```

### SPOP

　　**SPOP key**

　　移除并返回集合中的一个随机元素。

　　如果只想获取一个随机元素，但不想该元素从集合中被移除的话，可以使用 `SRANDMEMBER` 命令。

　　**可用版本：**

> = 1.0.0

　　**时间复杂度:**
O(1)

　　**返回值:**
| 被移除的随机元素。
| 当 `key` 不存在或 `key` 是空集时，返回 `nil` 。

　　::

```
redis> SMEMBERS db
1) "MySQL"
2) "MongoDB"
3) "Redis"

redis> SPOP db
"Redis"

redis> SMEMBERS db
1) "MySQL"
2) "MongoDB"

redis> SPOP db
"MySQL"

redis> SMEMBERS db
1) "MongoDB"
```

### SRANDMEMBER

　　**SRANDMEMBER key [count]**

　　如果命令执行时，只提供了 `key` 参数，那么返回集合中的一个随机元素。

　　从 Redis 2.6 版本开始， `SRANDMEMBER`_ 命令接受可选的 `count` 参数：

- 如果 `count` 为正数，且小于集合基数，那么命令返回一个包含 `count` 个元素的数组，数组中的元素\ **各不相同**\ 。如果 `count` 大于等于集合基数，那么返回整个集合。
- 如果 `count` 为负数，那么命令返回一个数组，数组中的元素\ **可能会重复出现多次**\ ，而数组的长度为 `count` 的绝对值。

　　该操作和 `SPOP` 相似，但 `SPOP` 将随机元素从集合中移除并返回，而 `SRANDMEMBER`_ 则仅仅返回随机元素，而不对集合进行任何改动。

　　**可用版本：**

> = 1.0.0

　　**时间复杂度:**
| 只提供 `key` 参数时为 O(1) 。
| 如果提供了 `count` 参数，那么为 O(N) ，N 为返回数组的元素个数。

　　**返回值:**
| 只提供 `key` 参数时，返回一个元素；如果集合为空，返回 `nil` 。
| 如果提供了 `count` 参数，那么返回一个数组；如果集合为空，返回空数组。

　　::

```
# 添加元素

redis> SADD fruit apple banana cherry
(integer) 3

# 只给定 key 参数，返回一个随机元素

redis> SRANDMEMBER fruit
"cherry"

redis> SRANDMEMBER fruit
"apple"

# 给定 3 为 count 参数，返回 3 个随机元素
# 每个随机元素都不相同

redis> SRANDMEMBER fruit 3
1) "apple"
2) "banana"
3) "cherry"

# 给定 -3 为 count 参数，返回 3 个随机元素
# 元素可能会重复出现多次

redis> SRANDMEMBER fruit -3
1) "banana"
2) "cherry"
3) "apple"

redis> SRANDMEMBER fruit -3
1) "apple"
2) "apple"
3) "cherry"

# 如果 count 是整数，且大于等于集合基数，那么返回整个集合

redis> SRANDMEMBER fruit 10
1) "apple"
2) "banana"
3) "cherry"

# 如果 count 是负数，且 count 的绝对值大于集合的基数
# 那么返回的数组的长度为 count 的绝对值

redis> SRANDMEMBER fruit -10
1) "banana"
2) "apple"
3) "banana"
4) "cherry"
5) "apple"
6) "apple"
7) "cherry"
8) "apple"
9) "apple"
10) "banana"

# SRANDMEMBER 并不会修改集合内容

redis> SMEMBERS fruit
1) "apple"
2) "cherry"
3) "banana"

# 集合为空时返回 nil 或者空数组

redis> SRANDMEMBER not-exists
(nil)

redis> SRANDMEMBER not-eixsts 10
(empty list or set)
```

### SREM

　　**SREM key member [member ...]**

　　移除集合 `key` 中的一个或多个 `member` 元素，不存在的 `member` 元素会被忽略。

　　当 `key` 不是集合类型，返回一个错误。

> 在 Redis 2.4 版本以前， `SREM`_ 只接受单个 `member` 值。

　　**可用版本：**

> = 1.0.0

　　**时间复杂度:**
O(N)， `N` 为给定 `member` 元素的数量。

　　**返回值:**
被成功移除的元素的数量，不包括被忽略的元素。

　　::

```
# 测试数据

redis> SMEMBERS languages
1) "c"
2) "lisp"
3) "python"
4) "ruby"


# 移除单个元素

redis> SREM languages ruby
(integer) 1


# 移除不存在元素

redis> SREM languages non-exists-language
(integer) 0


# 移除多个元素

redis> SREM languages lisp python c
(integer) 3

redis> SMEMBERS languages
(empty list or set)
```

### SSCAN

　　**SSCAN key cursor [MATCH pattern] [COUNT count]**

　　详细信息请参考 `SCAN` 命令。

### SUNION

　　**SUNION key [key ...]**

　　返回一个集合的全部成员，该集合是所有给定集合的并集。

　　不存在的 `key` 被视为空集。

　　**可用版本：**

> = 1.0.0

　　**时间复杂度:**
O(N)， `N` 是所有给定集合的成员数量之和。

　　**返回值:**
并集成员的列表。

　　::

```
redis> SMEMBERS songs
1) "Billie Jean"

redis> SMEMBERS my_songs
1) "Believe Me"

redis> SUNION songs my_songs
1) "Billie Jean"
2) "Believe Me"
```

### SUNIONSTORE

　　**SUNIONSTORE destination key [key ...]**

　　这个命令类似于 `SUNION` 命令，但它将结果保存到 `destination` 集合，而不是简单地返回结果集。

　　如果 `destination` 已经存在，则将其覆盖。

　　`destination` 可以是 `key` 本身。

　　**可用版本：**

> = 1.0.0

　　**时间复杂度:**
O(N)， `N` 是所有给定集合的成员数量之和。

　　**返回值:**
结果集中的元素数量。

　　::

```
redis> SMEMBERS NoSQL
1) "MongoDB"
2) "Redis"

redis> SMEMBERS SQL
1) "sqlite"
2) "MySQL"

redis> SUNIONSTORE db NoSQL SQL
(integer) 4

redis> SMEMBERS db
1) "MySQL"
2) "sqlite"
3) "MongoDB"
4) "Redis"
```

## ZSet（有序集合）

　　**有序集合为每一个元素设置一个分数（score）作为排序依据。**
有序集合的编码可以使ziplist或者skiplist

- ziplist编码的有序集合对象使用压缩列表作为底层实现，每个集合元素使用两个紧挨在一起的压缩列表节点来保存，第一个节点保存元素的成员，第二个节点保存元素的分值。并且压缩列表内的集合元素按分值从小到大的顺序进行排列，小的放置在靠近表头的位置，大的放置在靠近表尾的位置。
- skiplist编码的依序集合对象使用zset结构作为底层实现，一个zset结构同时包含一个字典和一个跳跃表

　　当有序结合对象同时满足以下两个条件时，对象使用ziplist编码，否则使用skiplist编码

- 保存的元素数量小于128
- 保存的所有元素长度都小于64字节

### ZADD

　　**ZADD key score member [[score member] [score member] ...]**

　　将一个或多个 `member` 元素及其 `score` 值加入到有序集 `key` 当中。

　　如果某个 `member` 已经是有序集的成员，那么更新这个 `member` 的 `score` 值，并通过重新插入这个 `member` 元素，来保证该 `member` 在正确的位置上。

　　`score` 值可以是整数值或双精度浮点数。

　　如果 `key` 不存在，则创建一个空的有序集并执行 `ZADD`_ 操作。

　　当 `key` 存在但不是有序集类型时，返回一个错误。

　　对有序集的更多介绍请参见 `sorted set <http://redis.io/topics/data-types#sorted-sets>`_ 。

> 在 Redis 2.4 版本以前， `ZADD`_ 每次只能添加一个元素。

　　**可用版本：**

> = 1.2.0

　　**时间复杂度:**
O(M*log(N))， `N` 是有序集的基数， `M` 为成功添加的新成员的数量。

　　**返回值:**
被成功添加的新成员的数量，不包括那些被更新的、已经存在的成员。

　　::

```
# 添加单个元素

redis> ZADD page_rank 10 google.com
(integer) 1


# 添加多个元素

redis> ZADD page_rank 9 baidu.com 8 bing.com
(integer) 2

redis> ZRANGE page_rank 0 -1 WITHSCORES
1) "bing.com"
2) "8"
3) "baidu.com"
4) "9"
5) "google.com"
6) "10"


# 添加已存在元素，且 score 值不变

redis> ZADD page_rank 10 google.com
(integer) 0

redis> ZRANGE page_rank 0 -1 WITHSCORES  # 没有改变
1) "bing.com"
2) "8"
3) "baidu.com"
4) "9"
5) "google.com"
6) "10"


# 添加已存在元素，但是改变 score 值

redis> ZADD page_rank 6 bing.com
(integer) 0

redis> ZRANGE page_rank 0 -1 WITHSCORES  # bing.com 元素的 score 值被改变
1) "bing.com"
2) "6"
3) "baidu.com"
4) "9"
5) "google.com"
6) "10"
```

### ZCARD

　　**ZCARD key**

　　返回有序集 `key` 的基数。

　　**可用版本：**

> = 1.2.0

　　**时间复杂度:**
O(1)

　　**返回值:**
| 当 `key` 存在且是有序集类型时，返回有序集的基数。
| 当 `key` 不存在时，返回 `0` 。

　　::

```
redis > ZADD salary 2000 tom    # 添加一个成员
(integer) 1

redis > ZCARD salary
(integer) 1

redis > ZADD salary 5000 jack   # 再添加一个成员
(integer) 1

redis > ZCARD salary
(integer) 2

redis > EXISTS non_exists_key   # 对不存在的 key 进行 ZCARD 操作
(integer) 0

redis > ZCARD non_exists_key
(integer) 0
```

### ZCOUNT

　　**ZCOUNT key min max**

　　返回有序集 `key` 中， `score` 值在 `min` 和 `max` 之间(默认包括 `score` 值等于 `min` 或 `max` )的成员的数量。

　　关于参数 `min` 和 `max` 的详细使用方法，请参考 `ZRANGEBYSCORE` 命令。

　　**可用版本：**

> = 2.0.0

　　**时间复杂度:**
O(log(N))， `N` 为有序集的基数。

　　**返回值:**
`score` 值在 `min` 和 `max` 之间的成员的数量。

　　::

```
redis> ZRANGE salary 0 -1 WITHSCORES    # 测试数据
1) "jack"
2) "2000"
3) "peter"
4) "3500"
5) "tom"
6) "5000"

redis> ZCOUNT salary 2000 5000          # 计算薪水在 2000-5000 之间的人数
(integer) 3

redis> ZCOUNT salary 3000 5000          # 计算薪水在 3000-5000 之间的人数
(integer) 2
```

### ZINCRBY

　　**ZINCRBY key increment member**

　　为有序集 `key` 的成员 `member` 的 `score` 值加上增量 `increment` 。

　　可以通过传递一个负数值 `increment` ，让 `score` 减去相应的值，比如 `ZINCRBY key -5 member` ，就是让 `member` 的 `score` 值减去 `5` 。

　　当 `key` 不存在，或 `member` 不是 `key` 的成员时， `ZINCRBY key increment member` 等同于 `ZADD key increment member` 。

　　当 `key` 不是有序集类型时，返回一个错误。

　　`score` 值可以是整数值或双精度浮点数。

　　**可用版本：**

> = 1.2.0

　　**时间复杂度:**
O(log(N))

　　**返回值:**
`member` 成员的新 `score` 值，以字符串形式表示。

```
redis> ZSCORE salary tom 
"2000"

redis> ZINCRBY salary 2000 tom   # tom 加薪啦！
"4000"
```

### ZINTERSTORE

　　**ZINTERSTORE destination numkeys key [key ...] [WEIGHTS weight [weight ...]] [AGGREGATE SUM|MIN|MAX]**

　　计算给定的一个或多个有序集的交集，其中给定 `key` 的数量必须以 `numkeys` 参数指定，并将该交集(结果集)储存到 `destination` 。

　　默认情况下，结果集中某个成员的 `score` 值是所有给定集下该成员 `score` 值之和.

　　关于 `WEIGHTS` 和 `AGGREGATE` 选项的描述，参见 `ZUNIONSTORE` 命令。

　　**可用版本：**

> = 2.0.0

　　**时间复杂度:**
O(N_K)+O(M_log(M))， `N` 为给定 `key` 中基数最小的有序集， `K` 为给定有序集的数量， `M` 为结果集的基数。

　　**返回值:**
保存到 `destination` 的结果集的基数。

　　::

```
redis > ZADD mid_test 70 "Li Lei"
(integer) 1
redis > ZADD mid_test 70 "Han Meimei"
(integer) 1
redis > ZADD mid_test 99.5 "Tom"
(integer) 1

redis > ZADD fin_test 88 "Li Lei"
(integer) 1
redis > ZADD fin_test 75 "Han Meimei"
(integer) 1
redis > ZADD fin_test 99.5 "Tom"
(integer) 1

redis > ZINTERSTORE sum_point 2 mid_test fin_test
(integer) 3

redis > ZRANGE sum_point 0 -1 WITHSCORES     # 显示有序集内所有成员及其 score 值
1) "Han Meimei"
2) "145"
3) "Li Lei"
4) "158"
5) "Tom"
6) "199"
```

### ZLEXCOUNT

　　**ZLEXCOUNT key min max**

```
When all the elements in a sorted set are inserted with the same score,
in order to force lexicographical ordering,
this command returns the number of elements in the sorted set 
at key with a value between min and max.
```

　　对于一个所有成员的分值都相同的有序集合键 `key` 来说，
这个命令会返回该集合中，
成员介于 `min` 和 `max` 范围内的元素数量。

```
The min and max arguments have the same meaning as described for `ZRANGEBYLEX` .
```

　　这个命令的 `min` 参数和 `max` 参数的意义和 `ZRANGEBYLEX` 命令的 `min` 参数和 `max` 参数的意义一样。

```
the command has a complexity of just O(log(N)) 
because it uses elements ranks (see ZRANK) to get an idea of the range.
Because of this there is no need to do a work proportional to the size of the range.
```

　　**可用版本：**

> = 2.8.9

　　**时间复杂度：**
O(log(N))，其中 N 为有序集合包含的元素数量。

　　O(log(N)) with N being the number of elements in the sorted set.

　　**返回值：**
整数回复：指定范围内的元素数量。

　　Integer reply: the number of elements in the specified score range.

```
redis> ZADD myzset 0 a 0 b 0 c 0 d 0 e
(integer) 5

redis> ZADD myzset 0 f 0 g
(integer) 2

redis> ZLEXCOUNT myzset - +
(integer) 7

redis> ZLEXCOUNT myzset [b [f
(integer) 5
```

### ZRANGE

　　**ZRANGE key start stop [WITHSCORES]**

　　返回有序集 `key` 中，指定区间内的成员。

　　其中成员的位置按 `score` 值递增(从小到大)来排序。

　　具有相同 `score` 值的成员按字典序(`lexicographical order <http://en.wikipedia.org/wiki/Lexicographical_order>`_ )来排列。

　　如果你需要成员按 `score` 值递减(从大到小)来排列，请使用 `ZREVRANGE` 命令。

　　| 下标参数 `start` 和 `stop` 都以 `0` 为底，也就是说，以 `0` 表示有序集第一个成员，以 `1` 表示有序集第二个成员，以此类推。
| 你也可以使用负数下标，以 `-1` 表示最后一个成员， `-2` 表示倒数第二个成员，以此类推。

　　| 超出范围的下标并不会引起错误。
| 比如说，当 `start` 的值比有序集的最大下标还要大，或是 `start > stop` 时， `ZRANGE`_ 命令只是简单地返回一个空列表。
| 另一方面，假如 `stop` 参数的值比有序集的最大下标还要大，那么 Redis 将 `stop` 当作最大下标来处理。

　　| 可以通过使用 `WITHSCORES` 选项，来让成员和它的 `score` 值一并返回，返回列表以 `value1,score1, ..., valueN,scoreN` 的格式表示。
| 客户端库可能会返回一些更复杂的数据类型，比如数组、元组等。

　　**可用版本：**

> = 1.2.0

　　**时间复杂度:**
O(log(N)+M)， `N` 为有序集的基数，而 `M` 为结果集的基数。

　　**返回值:**
指定区间内，带有 `score` 值(可选)的有序集成员的列表。

　　redis > ZRANGE salary 0 -1 WITHSCORES             # 显示整个有序集成员

1. "jack"
2. "3500"
3. "tom"
4. "5000"
5. "boss"
6. "10086"

　　redis > ZRANGE salary 1 2 WITHSCORES              # 显示有序集下标区间 1 至 2 的成员

1. "tom"
2. "5000"
3. "boss"
4. "10086"

　　redis > ZRANGE salary 0 200000 WITHSCORES         # 测试 end 下标超出最大下标时的情况

1. "jack"
2. "3500"
3. "tom"
4. "5000"
5. "boss"
6. "10086"

　　redis > ZRANGE salary 200000 3000000 WITHSCORES   # 测试当给定区间不存在于有序集时的情况
(empty list or set)

### ZRANGEBYLEX

　　**ZRANGEBYLEX key min max [LIMIT offset count]**

```
When all the elements in a sorted set 
are inserted with the same score,
in order to force lexicographical ordering,
this command returns all the elements in the sorted set at key 
with a value between min and max.
```

　　当有序集合的所有成员都具有相同的分值时，
有序集合的元素会根据成员的字典序（lexicographical ordering）来进行排序，
而这个命令则可以返回给定的有序集合键 `key` 中，
值介于 `min` 和 `max` 之间的成员。

```
If the elements in the sorted set have different scores,
the returned elements are unspecified.
```

　　如果有序集合里面的成员带有不同的分值，
那么命令返回的结果是未指定的（unspecified）。

```
The elements are considered to be ordered 
from lower to higher strings 
as compared byte-by-byte using the memcmp() C function.

Longer strings are considered greater than shorter strings 
if the common part is identical.
```

　　命令会使用 C 语言的 `memcmp()` 函数，
对集合中的每个成员进行逐个字节的对比（byte-by-byte compare），
并按照从低到高的顺序，
返回排序后的集合成员。
如果两个字符串有一部分内容是相同的话，
那么命令会认为较长的字符串比较短的字符串要大。

```
The optional LIMIT argument 
can be used to only get a range of the matching elements 
(similar to SELECT LIMIT offset, count in SQL).

Keep in mind that if offset is large,
the sorted set needs to be traversed for offset elements 
before getting to the elements to return,
which can add up to O(N) time complexity.
```

　　可选的 `LIMIT offset count` 参数用于获取指定范围内的匹配元素
（就像 SQL 中的 `SELECT LIMIT offset count` 语句）。
需要注意的一点是，
如果 `offset` 参数的值非常大的话，
那么命令在返回结果之前，
需要先遍历至 `offset` 所指定的位置，
这个操作会为命令加上最多 O(N) 复杂度。

```
How to specify intervals
---------------------------

Valid start and stop must start with ( or [ ,
in order to specify if the range item is respectively exclusive or inclusive. 

The special values of + or - for start and stop 
have the special meaning of positively infinite and negatively infinite strings,
so for instance the command `ZRANGEBYLEX myzset - +` 
is guaranteed to return all the elements in the sorted set,
if all the elements have the same score.
```

#### 如何指定范围区间

　　合法的 `min` 和 `max` 参数必须包含 `(` 或者 `[` ，
其中 `(` 表示开区间（指定的值不会被包含在范围之内），
而 `[` 则表示闭区间（指定的值会被包含在范围之内）。

　　特殊值 `+` 和 `-` 在 `min` 参数以及 `max` 参数中具有特殊的意义，
其中 `+` 表示正无限，
而 `-` 表示负无限。
因此，
向一个所有成员的分值都相同的有序集合发送命令 `ZRANGEBYLEX <zset> - +` ，
命令将返回有序集合中的所有元素。

```
Details on strings comparison
--------------------------------

关于字符串对比的细节
------------------------


Strings are compared as binary array of bytes.
Because of how the ASCII character set is specified, 
this means that usually this also have the effect of comparing normal ASCII characters in an obvious dictionary way. 
However this is not true if non plain ASCII strings are used 
(for example utf8 strings).
```

　　..
Redis 会将字符串看作是二进制字节数组（binary array of bytes）来进行对比，
在 ASCII 字符集上，
这种对比方式和普通的 ASCII 对比方式都可以产生字典序排列结果，
但对于非 ASCII 字符集（比如 utf8 字符集来说），
普通字符串排序就没办法产生字典序排列结果了。

　　..
However the user can apply a transformation to the encoded string
so that the first part of the element inserted in the sorted set
will compare as the user requires for the specific application.
For example
if I want to add strings that will be compared in a case-insensitive way,
but I still want to retrieve the real case when querying,
I can add strings in the following way:

```
::

    ZADD autocomplete 0 foo:Foo 0 bar:BAR 0 zap:zap
```

　　..
Because of the first normalized part in every element (before the colon character),
we are forcing a given comparison,
however after the range is queries using ZRANGEBYLEX
the application can display to the user
the second part of the string,
after the colon.

　　..
The binary nature of the comparison allows to use sorted sets as a general purpose index,
for example the first part of the element can be a 64 bit big endian number:
since big endian numbers have the most significant bytes in the initial positions,
the binary comparison will match the numerical comparison of the numbers.
This can be used in order to implement range queries on 64 bit values.
As in the example below,
after the first 8 bytes we can store the value of the element we are actually indexing.

　　**可用版本：**

> = 2.8.9

　　**时间复杂度：**
O(log(N)+M)，
其中 N 为有序集合的元素数量，
而 M 则是命令返回的元素数量。
如果 M 是一个常数（比如说，用户总是使用 `LIMIT` 参数来返回最先的 10 个元素），
那么命令的复杂度也可以看作是 O(log(N)) 。

　　..
O(log(N)+M) with N being the number of elements in the sorted set
and M the number of elements being returned.
If M is constant (e.g. always asking for the first 10 elements with LIMIT),
you can consider it O(log(N)).

　　**返回值：**
数组回复：一个列表，列表里面包含了有序集合在指定范围内的成员。

```
Array reply: list of elements in the specified score range.




redis> ZADD myzset 0 a 0 b 0 c 0 d 0 e 0 f 0 g
(integer) 7

redis> ZRANGEBYLEX myzset - [c
1) "a"
2) "b"
3) "c"

redis> ZRANGEBYLEX myzset - (c
1) "a"
2) "b"

redis> ZRANGEBYLEX myzset [aaa (g
1) "b"
2) "c"
3) "d"
4) "e"
5) "f"
```

### ZRANGEBYSCORE

　　**ZRANGEBYSCORE key min max [WITHSCORES] [LIMIT offset count]**

　　返回有序集 `key` 中，所有 `score` 值介于 `min` 和 `max` 之间(包括等于 `min` 或 `max` )的成员。有序集成员按 `score` 值递增(从小到大)次序排列。

　　具有相同 `score` 值的成员按字典序(`lexicographical order <http://en.wikipedia.org/wiki/Lexicographical_order>`_)来排列(该属性是有序集提供的，不需要额外的计算)。

　　可选的 `LIMIT` 参数指定返回结果的数量及区间(就像SQL中的 `SELECT LIMIT offset, count` )，注意当 `offset` 很大时，定位 `offset` 的操作可能需要遍历整个有序集，此过程最坏复杂度为 O(N) 时间。

　　| 可选的 `WITHSCORES` 参数决定结果集是单单返回有序集的成员，还是将有序集成员及其 `score` 值一起返回。
| 该选项自 Redis 2.0 版本起可用。

　　**区间及无限**

　　`min` 和 `max` 可以是 `-inf` 和 `+inf` ，这样一来，你就可以在不知道有序集的最低和最高 `score` 值的情况下，使用 `ZRANGEBYSCORE`_ 这类命令。

　　默认情况下，区间的取值使用\ `闭区间 <http://zh.wikipedia.org/wiki/%E5%8D%80%E9%96%93>`_ (小于等于或大于等于)，你也可以通过给参数前增加 *`_(_`* 符号来使用可选的\ _`_开区间 <http://zh.wikipedia.org/wiki/%E5%8D%80%E9%96%93>_` (小于或大于)。

　　举个例子：

　　::

```
ZRANGEBYSCORE zset (1 5
```

　　返回所有符合条件 `1 < score <= 5` 的成员，而

　　::

```
ZRANGEBYSCORE zset (5 (10
```

　　则返回所有符合条件 `5 < score < 10` 的成员。

　　**可用版本：**

> = 1.0.5

　　**时间复杂度:**
O(log(N)+M)， `N` 为有序集的基数， `M` 为被结果集的基数。

　　**返回值:**
指定区间内，带有 `score` 值(可选)的有序集成员的列表。

　　::

```
redis> ZADD salary 2500 jack                        # 测试数据
(integer) 0
redis> ZADD salary 5000 tom
(integer) 0
redis> ZADD salary 12000 peter
(integer) 0

redis> ZRANGEBYSCORE salary -inf +inf               # 显示整个有序集
1) "jack"
2) "tom"
3) "peter"

redis> ZRANGEBYSCORE salary -inf +inf WITHSCORES    # 显示整个有序集及成员的 score 值
1) "jack"
2) "2500"
3) "tom"
4) "5000"
5) "peter"
6) "12000"

redis> ZRANGEBYSCORE salary -inf 5000 WITHSCORES    # 显示工资 <=5000 的所有成员
1) "jack"
2) "2500"
3) "tom"
4) "5000"

redis> ZRANGEBYSCORE salary (5000 400000            # 显示工资大于 5000 小于等于 400000 的成员
1) "peter"
```

### ZRANK

　　**ZRANK key member**

　　返回有序集 `key` 中成员 `member` 的排名。其中有序集成员按 `score` 值递增(从小到大)顺序排列。

　　排名以 `0` 为底，也就是说， `score` 值最小的成员排名为 `0` 。

　　使用 `ZREVRANK` 命令可以获得成员按 `score` 值递减(从大到小)排列的排名。

　　**可用版本：**

> = 2.0.0

　　**时间复杂度:**
O(log(N))

　　**返回值:**
| 如果 `member` 是有序集 `key` 的成员，返回 `member` 的排名。
| 如果 `member` 不是有序集 `key` 的成员，返回 `nil` 。

　　::

```
redis> ZRANGE salary 0 -1 WITHSCORES        # 显示所有成员及其 score 值
1) "peter"
2) "3500"
3) "tom"
4) "4000"
5) "jack"
6) "5000"

redis> ZRANK salary tom                     # 显示 tom 的薪水排名，第二
(integer) 1
```

### ZREM

　　**ZREM key member [member ...]**

　　移除有序集 `key` 中的一个或多个成员，不存在的成员将被忽略。

　　当 `key` 存在但不是有序集类型时，返回一个错误。

> 在 Redis 2.4 版本以前， `ZREM`_ 每次只能删除一个元素。

　　**可用版本：**

> = 1.2.0

　　**时间复杂度:**
O(M*log(N))， `N` 为有序集的基数， `M` 为被成功移除的成员的数量。

　　**返回值:**
被成功移除的成员的数量，不包括被忽略的成员。

```
# 测试数据

redis> ZRANGE page_rank 0 -1 WITHSCORES
1) "bing.com"
2) "8"
3) "baidu.com"
4) "9"
5) "google.com"
6) "10"


# 移除单个元素

redis> ZREM page_rank google.com
(integer) 1

redis> ZRANGE page_rank 0 -1 WITHSCORES
1) "bing.com"
2) "8"
3) "baidu.com"
4) "9"


# 移除多个元素

redis> ZREM page_rank baidu.com bing.com
(integer) 2

redis> ZRANGE page_rank 0 -1 WITHSCORES
(empty list or set)


# 移除不存在元素

redis> ZREM page_rank non-exists-element
(integer) 0
```

### ZREMRANGEBYLEX

　　**ZREMRANGEBYLEX key min max**

```
When all the elements in a sorted set are inserted with the same score,
in order to force lexicographical ordering, 
this command removes all elements in the sorted set stored at key 
between the lexicographical range specified by min and max.
```

　　对于一个所有成员的分值都相同的有序集合键 `key` 来说，
这个命令会移除该集合中，
成员介于 `min` 和 `max` 范围内的所有元素。

```
The meaining of min and max are the same of the ZRANGEBYLEX command. 
Similarly, 
this command actually returns the same elements that ZRANGEBYLEX would return 
if called with the same min and max arguments.
```

　　这个命令的 `min` 参数和 `max` 参数的意义和 `ZRANGEBYLEX` 命令的 `min` 参数和 `max` 参数的意义一样。

　　**可用版本：**

> = 2.8.9

　　**时间复杂度：**
O(log(N)+M)，
其中 N 为有序集合的元素数量，
而 M 则为被移除的元素数量。
O(log(N)+M) with N being the number of elements in the sorted set and M the number of elements removed by the operation.

　　**返回值：**
整数回复：被移除的元素数量。

```
Integer reply: the number of elements removed.




redis> ZADD myzset 0 aaaa 0 b 0 c 0 d 0 e
(integer) 5

redis> ZADD myzset 0 foo 0 zap 0 zip 0 ALPHA 0 alpha
(integer) 5

redis> ZRANGE myzset 0 -1
1) "ALPHA"
2) "aaaa"
3) "alpha"
4) "b"
5) "c"
6) "d"
7) "e"
8) "foo"
9) "zap"
10) "zip"

redis> ZREMRANGEBYLEX myzset [alpha [omega
(integer) 6

redis> ZRANGE myzset 0 -1
1) "ALPHA"
2) "aaaa"
3) "zap"
4) "zip"
```

### ZREMRANGEBYRANK

　　**ZREMRANGEBYRANK key start stop**

　　移除有序集 `key` 中，指定排名(rank)区间内的所有成员。

　　区间分别以下标参数 `start` 和 `stop` 指出，包含 `start` 和 `stop` 在内。

　　| 下标参数 `start` 和 `stop` 都以 `0` 为底，也就是说，以 `0` 表示有序集第一个成员，以 `1` 表示有序集第二个成员，以此类推。
| 你也可以使用负数下标，以 `-1` 表示最后一个成员， `-2` 表示倒数第二个成员，以此类推。

　　**可用版本：**

> = 2.0.0

　　**时间复杂度:**
O(log(N)+M)， `N` 为有序集的基数，而 `M` 为被移除成员的数量。

　　**返回值:**
被移除成员的数量。

```
redis> ZADD salary 2000 jack
(integer) 1
redis> ZADD salary 5000 tom
(integer) 1
redis> ZADD salary 3500 peter
(integer) 1

redis> ZREMRANGEBYRANK salary 0 1       # 移除下标 0 至 1 区间内的成员
(integer) 2

redis> ZRANGE salary 0 -1 WITHSCORES    # 有序集只剩下一个成员
1) "tom"
2) "5000"
```

### ZREMRANGEBYSCORE

　　**ZREMRANGEBYSCORE key min max**

　　移除有序集 `key` 中，所有 `score` 值介于 `min` 和 `max` 之间(包括等于 `min` 或 `max` )的成员。

　　自版本2.1.6开始， `score` 值等于 `min` 或 `max` 的成员也可以不包括在内，详情请参见 `ZRANGEBYSCORE` 命令。

　　**可用版本：**

> = 1.2.0

　　**时间复杂度:**
O(log(N)+M)， `N` 为有序集的基数，而 `M` 为被移除成员的数量。

　　**返回值:**
被移除成员的数量。

　　redis> ZRANGE salary 0 -1 WITHSCORES          # 显示有序集内所有成员及其 score 值

1) "tom"
2) "2000"
3) "peter"
4) "3500"
5) "jack"
6) "5000"

```
redis> ZREMRANGEBYSCORE salary 1500 3500      # 移除所有薪水在 1500 到 3500 内的员工
(integer) 2

redis> ZRANGE salary 0 -1 WITHSCORES          # 剩下的有序集成员
1) "jack"
2) "5000"
```

### ZREVRANGE

　　**ZREVRANGE key start stop [WITHSCORES]**

　　返回有序集 `key` 中，指定区间内的成员。

　　| 其中成员的位置按 `score` 值递减(从大到小)来排列。
| 具有相同 `score` 值的成员按字典序的逆序(`reverse lexicographical order <http://en.wikipedia.org/wiki/Lexicographical_order#Reverse_lexicographic_order>`_)排列。

　　除了成员按 `score` 值递减的次序排列这一点外， `ZREVRANGE`_ 命令的其他方面和 `ZRANGE` 命令一样。

　　**可用版本：**

> = 1.2.0

　　**时间复杂度:**
O(log(N)+M)， `N` 为有序集的基数，而 `M` 为结果集的基数。

　　**返回值:**
指定区间内，带有 `score` 值(可选)的有序集成员的列表。

```
redis> ZRANGE salary 0 -1 WITHSCORES        # 递增排列
1) "peter"
2) "3500"
3) "tom"
4) "4000"
5) "jack"
6) "5000"

redis> ZREVRANGE salary 0 -1 WITHSCORES     # 递减排列
1) "jack"
2) "5000"
3) "tom"
4) "4000"
5) "peter"
6) "3500"
```

### ZREVRANGEBYSCORE

　　**ZREVRANGEBYSCORE key max min [WITHSCORES] [LIMIT offset count]**

　　返回有序集 `key` 中， `score` 值介于 `max` 和 `min` 之间(默认包括等于 `max` 或 `min` )的所有的成员。有序集成员按 `score` 值递减(从大到小)的次序排列。

　　具有相同 `score` 值的成员按字典序的逆序(`reverse lexicographical order <http://en.wikipedia.org/wiki/Lexicographical_order>`_ )排列。

　　除了成员按 `score` 值递减的次序排列这一点外， `ZREVRANGEBYSCORE`_ 命令的其他方面和 `ZRANGEBYSCORE` 命令一样。

　　**可用版本：**

> = 2.2.0

　　**时间复杂度:**
O(log(N)+M)， `N` 为有序集的基数， `M` 为结果集的基数。

　　**返回值:**
指定区间内，带有 `score` 值(可选)的有序集成员的列表。

```
redis > ZADD salary 10086 jack
(integer) 1
redis > ZADD salary 5000 tom
(integer) 1
redis > ZADD salary 7500 peter
(integer) 1
redis > ZADD salary 3500 joe
(integer) 1

redis > ZREVRANGEBYSCORE salary +inf -inf   # 逆序排列所有成员
1) "jack"
2) "peter"
3) "tom"
4) "joe"

redis > ZREVRANGEBYSCORE salary 10000 2000  # 逆序排列薪水介于 10000 和 2000 之间的成员
1) "peter"
2) "tom"
3) "joe"
```

### ZREVRANK

　　**ZREVRANK key member**

　　返回有序集 `key` 中成员 `member` 的排名。其中有序集成员按 `score` 值递减(从大到小)排序。

　　排名以 `0` 为底，也就是说， `score` 值最大的成员排名为 `0` 。

　　使用 `ZRANK` 命令可以获得成员按 `score` 值递增(从小到大)排列的排名。

　　**可用版本：**

> = 2.0.0

　　**时间复杂度:**
O(log(N))

　　**返回值:**
| 如果 `member` 是有序集 `key` 的成员，返回 `member` 的排名。
| 如果 `member` 不是有序集 `key` 的成员，返回 `nil` 。

```
redis 127.0.0.1:6379> ZRANGE salary 0 -1 WITHSCORES     # 测试数据
1) "jack"
2) "2000"
3) "peter"
4) "3500"
5) "tom"
6) "5000"

redis> ZREVRANK salary peter     # peter 的工资排第二
(integer) 1

redis> ZREVRANK salary tom       # tom 的工资最高
(integer) 0
```

### ZSCAN

　　**ZSCAN key cursor [MATCH pattern] [COUNT count]**

　　详细信息请参考 `SCAN` 命令。

### ZSCORE

　　**ZSCORE key member**

　　返回有序集 `key` 中，成员 `member` 的 `score` 值。

　　如果 `member` 元素不是有序集 `key` 的成员，或 `key` 不存在，返回 `nil` 。

　　**可用版本：**

> = 1.2.0

　　**时间复杂度:**
O(1)

　　**返回值:**
`member` 成员的 `score` 值，以字符串形式表示。

　　:

```
redis> ZRANGE salary 0 -1 WITHSCORES    # 测试数据
1) "tom"
2) "2000"
3) "peter"
4) "3500"
5) "jack"
6) "5000"

redis> ZSCORE salary peter              # 注意返回值是字符串
"3500"
```

### ZUNIONSTORE

　　**ZUNIONSTORE destination numkeys key [key ...] [WEIGHTS weight [weight ...]] [AGGREGATE SUM|MIN|MAX]**

　　计算给定的一个或多个有序集的并集，其中给定 `key` 的数量必须以 `numkeys` 参数指定，并将该并集(结果集)储存到 `destination` 。

　　默认情况下，结果集中某个成员的 `score` 值是所有给定集下该成员 `score` 值之 *和* 。

　　**WEIGHTS**

　　使用 `WEIGHTS` 选项，你可以为 *每个* 给定有序集 *分别* 指定一个乘法因子(multiplication factor)，每个给定有序集的所有成员的 `score` 值在传递给聚合函数(aggregation function)之前都要先乘以该有序集的因子。

　　如果没有指定 `WEIGHTS` 选项，乘法因子默认设置为 `1` 。

　　**AGGREGATE**

　　使用 `AGGREGATE` 选项，你可以指定并集的结果集的聚合方式。

　　默认使用的参数 `SUM` ，可以将所有集合中某个成员的 `score` 值之 *和* 作为结果集中该成员的 `score` 值；使用参数 `MIN` ，可以将所有集合中某个成员的 *最小*  `score` 值作为结果集中该成员的 `score` 值；而参数 `MAX` 则是将所有集合中某个成员的 *最大*  `score` 值作为结果集中该成员的 `score` 值。

　　**可用版本：**

> = 2.0.0

　　**时间复杂度:**
O(N)+O(M log(M))， `N` 为给定有序集基数的总和， `M` 为结果集的基数。

　　**返回值:**
保存到 `destination` 的结果集的基数。

```
redis> ZRANGE programmer 0 -1 WITHSCORES
1) "peter"
2) "2000"
3) "jack"
4) "3500"
5) "tom"
6) "5000"

redis> ZRANGE manager 0 -1 WITHSCORES
1) "herry"
2) "2000"
3) "mary"
4) "3500"
5) "bob"
6) "4000"

redis> ZUNIONSTORE salary 2 programmer manager WEIGHTS 1 3   # 公司决定加薪。。。除了程序员。。。
(integer) 6

redis> ZRANGE salary 0 -1 WITHSCORES
1) "peter"
2) "2000"
3) "jack"
4) "3500"
5) "tom"
6) "5000"
7) "herry"
8) "6000"
9) "mary"
10) "10500"
11) "bob"
12) "12000"
```
