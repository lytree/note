---
title: Set
date: 2022-10-30T13:22:31Z
lastmod: 2022-10-30T13:22:31Z
---

# Set

---

## HashSet 集合

> 底层数据结构是哈希表(是一个元素为链表的数组) + 红黑树

　　**要点**

- 实现 Set 接口
- 不保证迭代顺序
- 允许元素为 null
- 底层实际上是一个 HashMap 实例
- 非同步
- 初始容量非常影响迭代性能

　　HashSet 实际上就是封装了 HashMap，操作 HashSet 元素实际上就是操作 HashMap。这也是面向对象的一种体现，重用性贼高！

　　详解见 [HashMap](map.md##HashMap)

## TreeSet 集合

> 底层数据结构是红黑树(是一个自平衡的二叉树)

　　保证元素的排序方式

　　**要点**

- 实现NavigableSet接口
- 可以实现排序功能
- 底层实际上是一个TreeMap实例
- 非同步

## LinkedHashSet 集合

> 底层数据结构由哈希表(是一个元素为链表的数组)和双向链表组成。
