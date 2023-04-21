---
title: List
date: 2022-10-30T13:22:01Z
lastmod: 2022-10-30T13:22:01Z
---

# List

---

　　**List 集合的三个子类：**

- ArrayList :底层数据结构是数组。线程不安全
- LinkedList :底层数据结构是链表。线程不安全
- Vector:底层数据结构是数组。线程安全

## ArrayList

### 简介

- ArrayList 继承了 AbstractList，实现了 List。它是一个数组队列，提供了相关的添加、删除、修改、遍历等功能。
- ArrayList 实现了 RandomAccess 接口， RandomAccess 是一个标志接口，表明实现这个这个接口的 List 集合是支持快速随机访问的。在 ArrayList 中，我们即可以通过元素的序号快速获取元素对象，这就是快速随机访问。
- ArrayList 实现了 Cloneable 接口，即覆盖了函数 clone()，能被克隆。
- ArrayList 实现 java.io.Serializable 接口，这意味着 ArrayList 支持序列化，能通过序列化去传输。
- 和 Vector 不同，ArrayList 中的操作不是线程安全的！所以，建议在单线程中才使用 ArrayList，而在多线程中可以选择 Vector 或者 CopyOnWriteArrayList。

### System.arraycopy()和 Arrays.copyOf()方法

#### 两者联系与区别

　　**联系：**

　　看两者源代码可以发现 copyOf()内部调用了 System.arraycopy()方法

　　**区别：**

　　arraycopy()需要目标数组，将原数组拷贝到你自己定义的数组里，而且可以选择拷贝的起点和长度以及放入新数组中的位置

　　copyOf()是系统自动在内部新建一个数组，并返回该数组。

　　**注意的是：**

- java 中的 length 属性是针对数组说的,比如说你声明了一个数组,想知道这个数组的长度则用到了 length 这个属性.
- java 中的 length()方法是针对字 符串 String 说的,如果想看这个字符串的长度则用到 length()这个方法.
- java 中的 size()方法是针对泛型集合说的,如果想看这个泛型有多少个元素,就调用此方法来查看!

### 细节

- ArrayList 是基于动态数组实现的，在增删时候，需要数组的拷贝复制。(navite 方法由 C/C++实现)
- ArrayList 的默认初始化容量是 10，每次扩容时候增加原先容量的一半，也就是变为原来的 1.5 倍
- 删除元素时不会减少容量，若希望减少容量则调用 trimToSize()
- 它不是线程安全的。它能存放 null 值。

## LinkedList

### 简介

> 底层实现是双向链表(双向链表方便实现往前遍历)
> LinkedList 是一个实现了 List 接口和 Deque 接口的双端链表。 LinkedList 底层的链表结构使它支持高效的插入和删除操作，另外它实现了 Deque 接口，使得 LinkedList 类也具有队列的特性; LinkedList 不是线程安全的，如果想使 LinkedList 变成线程安全的，可以调用静态类 Collections 类中的 synchronizedList 方法：

## Vector：

- 底层是数组，现在已少用，被 ArrayList 替代，原因有两个：
- Vector 所有方法都是同步，有性能损失。
- Vector 初始 length 是 10 超过 length 时 以 100%比率增长，相比于 ArrayList 更多消耗内存。
- ArrayList 在底层数组不够用时在原来的基础上扩展 0.5 倍，Vector 是扩展 1 倍。

　　**总的来说：查询多用 ArrayList，增删多用 LinkedList。**

　　ArrayList 增删慢不是绝对的(在数量大的情况下，已测试)：

　　如果增加元素一直是使用 add()(增加到末尾)的话，那是 ArrayList 要快

　　一直删除末尾的元素也是 ArrayList 要快【不用复制移动位置】

　　至于如果删除的是中间的位置的话，还是 ArrayList 要快！

## SkipList
