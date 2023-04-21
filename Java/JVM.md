---
title: JVM
date: 2022-10-30T13:39:04Z
lastmod: 2022-10-30T13:39:04Z
---

# JVM

## JVM四种引用

### 强引用

　　特点：GC时，永远不会被回收
使用场景

> new 对象

### 软引用SoftReference<Object>(obj);

　　特点：内存不足时（自动触发GC），会被回收
使用场景

> 缓存

### 弱引用WeakReference<Object>(obj)

　　特点：无论内存是否充足，只要进行GC，都会被回收
使用场景

> 一次性对象

### 虚引用PhantomReference<>(new Object(),new ReferenceQueue<>())

　　特点：如同虚设，和没有引用没什么区别
使用场景

> 1. 管理堆外面的引用

## java 内存区域

　　jdk8 之前：
![](https://cdn.nlark.com/yuque/0/2020/png/381674/1592902120990-cffe2ea4-a4ed-4505-8bbd-c9b64730ae6e.png#crop=0&crop=0&crop=1&crop=1&height=651&id=Hp6uW&originHeight=651&originWidth=694&originalType=binary&ratio=1&rotation=0&showTitle=false&size=0&status=done&style=none&title=&width=694)

　　jdk8 及之后

　　![](https://cdn.nlark.com/yuque/0/2020/png/381674/1592902119376-b0ab487a-f7f3-454f-a784-aad737462ad4.png#crop=0&crop=0&crop=1&crop=1&height=651&id=pLy1v&originHeight=651&originWidth=694&originalType=binary&ratio=1&rotation=0&showTitle=false&size=0&status=done&style=none&title=&width=694)

### 程序计数器作用：（唯一）

- 字节码解释器通过改变程序计数器来依次读取指令，从而实现对代码得流程控制。
- 多线程情况下，程序计数器用于记录当前线程执行的位置，以便当线程切换回来时能够知道线程上次运行到的位置
- 计数器记录的是正在执行的虚拟机字节码指令的地址；如果正在执行的是本地（Native）方法，这个计数器值则应为空（Undefined）

### Java 虚拟机栈

　　为虚拟机执行 Java 方法 （也就是字节码）服务

　　java 虚拟机栈是线程私有的，他与线程的声明周期同步。虚拟机栈描述的是 java 方法执行的内存模型，每个方法执行都会创建一个栈帧，栈帧包含**局部变量表、操作数栈、动态连接、方法出口**等。
局部变量表中存储的是基础数据类型，对象引用（reference（引用）类型，它并不等同于对象本身，可能是一个指向对象起始地址的引用指针，也可能是指向一个代表对象的句柄或者其他与此对象相关的位置）和returnAddress 类型（指向了一条字节码指令的地址）

　　`Slot（变量槽）`是局部变量表中最基本的存储单元（变量槽的大小不固定，具体由虚拟机自行实现）

　　在局部变量表中的存储空间以局部变量槽（Slot）来表示，其中64位长度的long和double类型的数据会占用两个变量槽，其余的数据类型只占用一个。当进入一个方法后，栈帧中需要分配多大的局部变量空间已经固定（变量槽数量）

### 本地方法栈

　　为虚拟机使用到的 Native 方法服务 （一个 Native Method 就是一个 java 调用非 java 代码的接口）

### 堆

　　**存放几乎所有实例对象**
从 jdk 1.7 开始已经默认开启逃逸分析，如果某些方法中的对象引用没有被返回或者未被外面使用（也就是未逃逸出去），那么对象可以直接在栈上分配内存。

　　java 堆是垃圾收集器管理的主要区域（GC 堆），在细分为新生代，老年代，永生代
JDK8 后方法区（永生代）移除，用元空间代替，元空间使用直接内存

### 方法区（方法区也被称为永久代）

　　用于存储已被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等数据

> 方法区和永久代的关系
> 永久代是 HotSpot 的概念，方法区是 Java 虚拟机规范中的定义，是一种规范，而永久代是一种实现，一个是标准一个是实现，其他的虚拟机实现并没有永久代这一说法

### 为何永久代被元空间替代

- 永久代有固定上限，无法进行调整，而元空间直接使用内存，受本机可用内存限制。
- 元空间里存放的是类的元数据，加载多少类的元数据有系统的四季可用空间来控制，可以加载更多的类
- JDK8 合并 HotSpot 和 JRockit 代码时，jrockit 不存在永久代，所以合并后没必要设置一个永生代位置

### 运行时常量池（用于存放编译期生成的各种字面量和符号引用）

　　Jdk7 之前常量池包含字符常量池存放在方法去
Jdk7 字符串常量池在堆中，运行时常量池剩余东西还在方法区
Jdk8 移除方法区，字符串常量池还在堆中，但运行时常量池在元空间中

## 对象创建过程

　　![](https://cdn.nlark.com/yuque/__mermaid_v3/4122e8fc2d6e2995074ede61ff44aef4.svg#lake_card_v2=eyJ0eXBlIjoibWVybWFpZCIsImNvZGUiOiJncmFwaCBMUjtcbiAgICBBKOexu-WKoOi9vSkgLS0-IEIo5YiG6YWN5YaF5a2YKSAtLT4gQyjliJ3lp4vljJbpm7blgLwpIC0tPkYo6K6-572u5a-56LGh5aS0KS0tPiBFKOaJp-ihjGluaXTmlrnms5UpIiwidXJsIjoiaHR0cHM6Ly9jZG4ubmxhcmsuY29tL3l1cXVlL19fbWVybWFpZF92My80MTIyZThmYzJkNmUyOTk1MDc0ZWRlNjFmZjQ0YWVmNC5zdmciLCJpZCI6ImYwMjBlYmMzIiwiY2FyZCI6ImRpYWdyYW0ifQ==)

### 类加载检查

　　![](https://cdn.nlark.com/yuque/0/2020/png/381674/1592902119321-dda9e910-093f-47bf-890e-4719cb145075.png#crop=0&crop=0&crop=1&crop=1&height=291&id=vJmAT&originHeight=291&originWidth=700&originalType=binary&ratio=1&rotation=0&showTitle=false&size=0&status=done&style=none&title=&width=700)

- 加载
  - 将 class 文件加载到内存中
  - 将静态数据转变为方法区（元空间）运行时的数据结构
  - 在堆中生成 Class 类作为数据访问入口
- 链接
  - 验证：安全性检查
  - 准备：将静态变量在方法区分配空间，并初始化赋值（不包括实例变量）
  - 解析：虚拟机将符号引用转变为直接引用（符号引用比如`import java.util.ArrayList`）
- 初始化

### 分配内存：

　　分配方式：**指针碰撞**和**空闲列表**

- 指针碰撞：假设java内存绝对规整，被使用的和未使用的中间放一个指针作为分界点的指示器，分配内存时，仅需把指针挪动挪动一段和对象大小相同的距离即可。
- 空闲列表：虚拟机维护一个列表，记录那些内存块是可用的，分配时从列表中获取一块足够大的空间去分配，并更新列表。

> 使用Serial，ParNew等压缩整理整理过程的收集器是，系统采用指针碰撞；而是用CMS等交换算法收集器时，理论上采用空闲列表分配空间

　　内存分配（指针碰撞存在并发安全）并发问题：**CAS+重试**或**TLAB**

- CAS+重试：
- TLAB（本地线程分配缓存）：把内存分配的动作按照线程划分在不同的空间之中进行，每个线程在JAVA堆中预先分配小块内存，只有本地缓存用完，分配新的缓存区时才需要同步锁定。

> 内存分配完成，虚拟机必须将已分配的内存空间初始化零值，如果使用TLAB，初始化零值也可以提前至TLAB分配时顺便初始化零值

### 设置对象头

### 执行Init方法

## 对象卸载

　　Java虚拟机自带的类加载器所加载的类，在整个虚拟机的生命周期中，都不会被卸载。
用户自定义的类加载器是可以被卸载的。
**卸载时机：**

1. 该类的所有实例对象都被回收
2. 该类的类加载器对象已经被回收
3. 该类对应的Class对象没有被引用，无法在任何地方通过反射获取

## 类加载器的加载顺序

- 启动类加载器 `BootStrap ClassLoader`：rt.jar
- 扩展类加载器 `Extention ClassLoader`: 加载扩展的 jar 包
- 应用类加载器 `App ClassLoader`：指定的 classpath 下面的 jar 包
- 自定义类加载器 `Custom ClassLoader`：自定义的类加载器

## 双亲委派机制

　　当一个类收到加载请求时，不会先自己去尝试加载，会先委派给父类去完成（除了顶层的启动类加载器），只有父类加载器无法完成该请求时，子类加载器才会自行尝试自己加载
**继承ClassLoader，重写findClass可以破坏双亲委派机制**

##
