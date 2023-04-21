---
title: Java Stream
date: 2022-10-30T13:27:17Z
lastmod: 2022-10-30T13:27:17Z
---

# Java Stream

### Stream

> Stream（流）是一个来自数据源的元素队列并支持聚合操作

- 元素是特定类型的对象，形成一个队列。 Java中的Stream并不会存储元素，而是按需计算。
- **数据源** 流的来源。 可以是集合，数组，I/O channel， 产生器generator 等。
- **聚合操作** 类似SQL语句一样的操作， 比如filter, map, reduce, find, match, sorted等。

　　Stream还有两个特征

- **Pipelining**: 中间操作都会返回流对象本身。 这样多个操作可以串联成一个管道， 如同流式风格（fluent style）。 这样做可以对操作进行优化， 比如延迟执行(laziness)和短路( short-circuiting)。
- **内部迭代**： 以前对集合遍历都是通过Iterator或者For-Each的方式, 显式的在集合外部进行迭代， 这叫做外部迭代。 Stream提供了内部迭代的方式， 通过访问者模式(Visitor)实现。

### 生成流

- stream() -为集合创建串行流。
- parallelStream() − 为集合创建并行流。
- `Stream.of()` 从一堆对象中创建 Stream 流

### 数据处理

#### map

> 用于映射每个元素到对应的结果，可对当前元素进行数据处理

#### flatmap

> 将流的每个元素, 转换为其他对象的流,入参接受一个返回对象流的函数

#### filter（lamda表达式）

> 用于通过设置的条件过滤出元素(过滤条件对应的是留下的元素)

#### limit

> 用于获取指定数量的流

#### sorted

> 用于对流进行排序，可以传入**Comparator**参数控制排序顺序

#### distinct

> 用于消除流中的重复元素

#### isPrime()

> 用于检测是否是质数，是留下该元素

### 结束操作

#### forEach(Consumer)

> 遍历迭代流中的每个元素，无返回值

#### `forEachOrdered(Consumer)`

> 确保按照原始流的顺序执行。

#### collect(Collector)

> 使用 `Collector` 收集流元素到结果集合中

#### `collect(Supplier, BiConsumer, BiConsumer)`

> 收集流元素到结果集合中，第一个参数用于**创建一个新的结果集合**，第二个参数用于**将下一个元素加入到现有结果合集中**，第三个参数用于**将两个结果合集合并**

## 匹配

#### `allMatch(Predicate)` ：

> 如果流的每个元素根据提供的 `Predicate` 都返回 true 时，最终结果返回为 true。这个操作将会在第一个 false 之后短路，也就是不会在发生 false 之后继续执行计算。

#### `anyMatch(Predicate)`：

> 如果流中的任意一个元素根据提供的 `Predicate` 返回 true 时，最终结果返回为 true。这个操作将会在第一个 true 之后短路，也就是不会在发生 true 之后继续执行计算。

#### `noneMatch(Predicate)`：

> 如果流的每个元素根据提供的 `Predicate` 都返回 false 时，最终结果返回为 true。这个操作将会在第一个 true 之后短路，也就是不会在发生 true 之后继续执行计算。
