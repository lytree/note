---
title: B树
date: 2022-10-30T10:49:14Z
lastmod: 2022-10-30T10:49:14Z
---

# B树

## B 树（B-树）

### 定义

- 根节点至少有两个子节点；
- 每个中间节点都包含 k-1 个元素和 k 个孩子：$\lceil m/2 \rceil \leq k \leq m$
- 每一个叶子节点都包含 k-1 个元素：$\lceil m/2 \rceil  \leq k \leq m$
- 所有叶子节点都位于同一层
- 每个节点中的元素从小到大排列，节点中 k-1 个元素正好是 k 个孩子包含的元素值域划分

　　**例如**

　　![](assets/net-img-NIKEYn-20221030105009-qtc7vqs.jpg)[https://imgchr.com/i/NIKEYn](https://imgchr.com/i/NIKEYn)

## B+树

### 定义（与 B 树不同的）

- B+非叶子结点不保存关键字记录的内容指针，只进行数据索引
- B+树叶子结点保存父节点的所有关键字记录的内容指针，所有数据内容地址必须到叶子结点才能获取到。
- B+树叶子结点的关键字从小到大有序排序，左边结尾数据都会保存右边节点开始数据的指针
- 飞叶子节点的子节点数=关键字数
  **例子**
  ![](assets/net-img-NILhqg-20221030105009-gkjt4x4.jpg)[https://imgchr.com/i/NILhqg](https://imgchr.com/i/NILhqg)

## B*树

- 是 B+树的变体，在 B+树的非根和非叶子结点再增加指向兄弟的指针；
