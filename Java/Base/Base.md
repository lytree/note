---
title: Base
date: 2022-10-30T13:24:18Z
lastmod: 2022-10-30T13:24:18Z
---

# Base

## String 类和常量池

　　String 对象的两种创建方式：

```java
String str1 = "abcd";//先检查字符串常量池中有没有"abcd"，如果字符串常量池中没有，则创建一个，然后 str1 指向字符串常量池中的对象，如果有，则直接将 str1 指向"abcd""；
String str2 = new String("abcd");//堆中创建一个新的对象
String str3 = new String("abcd");//堆中创建一个新的对象
System.out.println(str1==str2);//false
System.out.println(str2==str3);//false
```

　　这两种不同的创建方法是有差别的。

　　第一种方式是在常量池中拿对象,如果没有则在字符串常量池中创建一个；
第二种方式是直接在堆内存空间创建一个新的对象。
**注意**

- 直接使用双引号声明出来的 String 对象会直接存储在常量池中
- 如果不是用双引号声明的 String 对象，可以使用 String 提供的 intern 方法，JDK1.7之前（不包含1.7）的处理方式是在常量池中创建与此 String 内容相同的字符串，并返回常量池中创建的字符串的引用，JDK1.7以及之后的处理方式是在常量池中记录此字符串的引用，并返回该引用

## StringBuffer 和 StrinBuilder 区别 ,String 为何不可变

　　java8 中 String 类使用 final 关键字修饰**字符数组**来保存字符串`private final char value[]`
java8 之后 java9 等 String 类使用 final 关键字修饰**字节数组**来保存字符串`private final byte[] value`

　　`StringBuffer` 和 `StringBuilder` 继承 `AbstractStringBuilder` 而在`AbstractStringBuilder`中使用字符数组存储字符串`char[] value`并没有使用 final 关键字修饰

- 从线程安全性考虑:
  String 中对象不可变,可理解为常量,线程安全.`AbstractStringBuilder`是公共父类,定义了一些字符串基本操作.Stringbuffer 对方法添加了同步锁或对调用的方法加了同步锁,线程安全.Stringbuilder 并没有对方法加同步锁加同步锁,所以非线程安全
- 从性能考虑:
  每次改变 String 类型会新建一个 String 对象,让后指针指向新的 String 对象,StringBuffer 每次使用对 StringBuffer 对象本身操作,不会产生新的对象,而是改变对象的引用.而 StringBuilder 相比 StringBuffer 性能会提升一部分,但需要承担线程不安全的风险

## 在 Java 中定义一个不做事且没有参数的构造方法的作用

　　在执行子类构造方法时,若没有调用`super()`,则默认会调用父类的无参构造函数,若父类中没有无参构造函数,编译就会报错

## 接口和抽象类的区别是什么？

1. 接口的方法默认是 public，所有方法在接口中不能有实现(Java 8 开始接口方法可以有默认实现），而抽象类可以有非抽象的方法。
2. 接口中除了 static、final 变量，不能有其他变量，而抽象类中则不一定。
3. 一个类可以实现多个接口，但只能实现一个抽象类。接口自己本身可以通过 extends 关键字扩展多个接口。
4. 接口方法默认修饰符是 public，抽象方法可以有 public、protected 和 default 这些修饰符（抽象方法就是为了被重写所以不能使用 private 关键字修饰！）。
5. 从设计层面来说，抽象是对类的抽象，是一种模板设计，而接口是对行为的抽象，是一种行为的规范。
6. 关于抽象类 JDK 1.8 以前，抽象类的方法默认访问权限为 protected JDK 1.8 时，抽象类的方法默认访问权限变为 default
7. JDK 1.8 以前，接口中的方法必须是 public 的 JDK 1.8 时，接口中的方法可以是 public 的，也可以是 default 的

## 成员变量与局部变量区别

　　语法上:成员变量可以被`public``private``static`等修饰,局部变量不能被修饰.但都能被`final`所修饰
内存中的存储方式:如果成员变量是使用`static`修饰的，那么这个成员变量是属于类的，如果没有使用`static`修饰，这个成员变量是属于实例的。而对象存在于堆内存，局部变量则存在于栈内存
生命周期:成员变量是对象的一部分，它随着对象的创建而存在，而局部变量随着方法的调用而自动消失
成员变量如果没有被赋初值,则会自动以类型的默认值而赋值（一种情况例外:被 final 修饰的成员变量也必须显式地赋值），而局部变量则不会自动赋值。

## 为什么重写 equals 时必须重写 hashCode 方法

　　`==` : 它的作用是判断两个对象的地址是不是相等。即，判断两个对象是不试同一个对象。

　　`equals()` : 它的作用也是判断两个对象是否相等。但它一般有两种使用情况(前面第 1 部分已详细介绍过)：
情况 1，类没有覆盖`equals()`方法。则通过`equals()`比较该类的两个对象时，等价于通过“==”比较这两个对象。
情况 2，类覆盖了`equals()`方法。一般，我们都覆盖`equals()`方法来两个对象的内容相等；若它们的内容相等，则返回 true(即，认为这两个对象相等)。
分两种情况

- 不会创建“类对应的散列表” 例如:不会创建该类的 HashSet 集合。

　　在这种情况下`equals()`用来比较该类的两个对象是否相等。而 hashCode() 则根本没有任何作用，所以，不用理会 hashCode()。

- 会创建“类对应的散列表” 例如，会创建该类的 HashSet 集合,且自定义对象为键值

1. 如果两个对象相等，那么它们的 hashCode()值一定相同。这里的相等是指，通过 equals()比较两个对象时返回 true。
2. 如果两个对象 hashCode()相等，它们并不一定相等。因为在散列表中，hashCode()相等，即两个键值对的哈希值相等。然而哈希值相等，并不一定能得出键值对相等。**“两个不同的键值对，哈希值相等”，这就是哈希冲突。**

　　散列表存储的是键值对(key-value)，它的特点是：能根据“键”快速的检索出对应的“值”。这其中就利用到了散列码！（可以快速找到所需要的对象）,如果不重写 hashcode 可能会导致相同含义的的不同对象被(hashcode 应该相等)pass 掉.而如果只重写 hashCode 不重写 equals 方法，那么 equals 只是判断两个对象是否是同一个对象

## 值传递和引用传递(java 是值传递)

　　值传递（pass by value）是指在调用函数时将实际参数复制一份传递到函数中，这样在函数中如果对参数进行修改，将不会影响到实际参数。
引用传递（pass by reference）是指在调用函数时将实际参数的地址直接传递到函数中，那么在函数中对参数所进行的修改，将影响到实际参数。

## Java 中的异常处理

　　**注意：异常和错误的区别：异常能被程序本身处理，错误是无法处理。**

- try 块： 用于捕获异常。其后可接零个或多个 catch 块，如果没有 catch 块，则必须跟一个 finally 块。
- catch 块： 用于处理 try 捕获到的异常。
- finally 块： 无论是否捕获或处理异常，finally 块里的语句都会被执行。当在 try 块或 catch 块中遇到 return 语句时，finally 语句块将在方法返回之前被执行。
  **在以下 4 种特殊情况下，finally 块不会被执行：**
- 在 finally 语句块第一行发生了异常。 因为在其他行，finally 块还是会得到执行
- 在前面的代码中用了 System.exit(int)已退出程序。 exit 是带参函数 ；若该语句在异常语句之后，finally 会执行
- 程序所在的线程死亡。
- 关闭 CPU。

> 如果 try 语句里有 return，返回的是 try 语句块中变量值。
> 详细执行过程如下：
> a.如果有返回值，就把返回值保存到局部变量中；
> b.执行 jsr 指令跳到 finally 语句里执行；
> c.执行完 finally 语句后，返回之前保存在局部变量表里的值。
> 如果 try，finally 语句里均有 return，忽略 try 的 return，而使用 finally 的 return.

## Java 中 IO 流分为几种?

　　按照流的流向分，可以分为输入流和输出流；
按照操作单元划分，可以划分为字节流和字符流；
按照流的角色划分为节点流和处理流。

## java 异常处理

- try 块： 用于捕获异常。其后可接零个或多个 catch 块，如果没有 catch 块，则必须跟一个 finally 块。
- catch 块： 用于处理 try 捕获到的异常。
- finally 块： 无论是否捕获或处理异常，finally 块里的语句都会被执行。当在 try 块或 catch 块中遇到 return 语句时，finally 语句块将在方法返回之前被执行。

　　在以下 4 种特殊情况下，finally 块不会被执行：

- 在 finally 语句块第一行发生了异常。 因为在其他行，finally 块还是会得到执行
- 在前面的代码中用了 System.exit(int)已退出程序。 exit 是带参函数 ；若该语句在异常语句之后，finally 会执行
- 程序所在的线程死亡。
- 关闭 CPU。
  **注意： 当 try 语句和 finally 语句中都有 return 语句时，在方法返回之前，finally 语句的内容将被执行，并且 finally 语句的返回值将会覆盖原始的返回值**

## Java修饰符

### 访问修饰符

#### public

#### protected

#### private

#### default

### 非访问修饰符

#### final 关键字

- final 修饰的类不能被继承，final 类中的所有成员方法都会被隐式的指定为 final 方法；
- final 修饰的方法不能被重写；
- final 修饰的变量是常量，如果是基本数据类型的变量，则其数值一旦在初始化之后便不能更改；如果是引用类型的变量，则在对其初始化之后便不能让其指向另一个对象。

#### static 关键字

- 修饰成员变量和成员方法: 被 static 修饰的成员属于类，不属于单个这个类的某个对象，被类中所有对象共享，可以并且建议通过类名调用。被 static 声明的成员变量属于静态成员变量，静态变量 存放在 Java 内存区域的方法区。
- 静态代码块: 静态代码块定义在类中方法外, 静态代码块在非静态代码块之前执行(静态代码块—>非静态代码块—>构造方法)。 该类不管创建多少对象，静态代码块只执行一次.
- 静态内部类（static 修饰类的话只能修饰内部类）： 静态内部类与非静态内部类之间存在一个最大的区别: **非静态内部类**在编译完成之后会隐含地保存着一个引用，该引用是指向创建它的外围类，但是**静态内部类**却没有。没有这个引用就意味着：1. 它的创建是不需要依赖外围类的创建。2. 它不能使用任何外围类的非 static 成员变量和方法。

#### abstract

#### synchronized

## Object类

### Object类所有方法详解

　　Object中含有： registerNatives()、getClass()、hashCode()、equals()、clone()、toString()、notify()、notifyAll()、wait(long)、wait(long,int)、wait()、finalize()

- registerNatives()
  - **注册 native方法**
- getClass()[(1)](#oJQGR)
  - 获取类的`Class` 对象
- hashCode()
- equals()
- clone()
- toString()
- wait()/ wait(long)/ wait(long,int)
  - 作用是阻塞当前线程，等待其他线程调用 notify()/notifyAll()方法将其唤醒。
  - 注意：
    - 此方法只能在当前线程获取到对象的锁监视器之后才能调用，否则会抛出 IllegalMonitorStateException异常。
    - 调用 wait方法，线程会将锁监视器进行释放；而 Thread.sleep，Thread.yield()并不会释放锁。
    - wait方法会一直阻塞，直到其他线程调用当前对象的 notify()/notifyAll()方法将其唤醒；而 wait(long)是等待给定超时时间内（单位毫秒），如果还没有调用 notify()/nofiyAll()会自动唤醒； wait(long,int)如果第二个参数大于 0并且小于 999999，则第一个参数 +1作为超时时间
- notify()/notifyAll()[(2)](#W03CQ)
  - 调用 `wait`方法，将锁释放并阻塞；这时另一个线程获取到了此对象锁，并调用此对象的 `notify()/notifyAll()`方法将之前的线程唤醒。
  - **注意：**
    - 调用 notify()后，阻塞线程被唤醒，可以参与锁的竞争，但可能调用 notify()方法的线程还要继续做其他事，锁并未释放，所以我们看到的结果是，无论 notify()是在方法一开始调用，还是最后调用，阻塞线程都要等待当前线程结束才能开始。
- finalize()
  - 此方法是在垃圾回收之前，JVM会调用此方法来清理资源。此方法可能会将对象重新置为可达状态，导致JVM无法进行垃圾回收。
  - finalize()方法具有如下4个特点：
    - 永远不要主动调用某个对象的 finalize()方法，该方法由垃圾回收机制自己调用；
    - finalize()何时被调用，是否被调用具有不确定性；
    - 当 JVM执行可恢复对象的 finalize()可能会将此对象重新变为可达状态；
    - 当 JVM执行 finalize()方法时出现异常，垃圾回收机制不会报告异常，程序继续执行。

### 

## 类型转换规则

### 自动类型转换

- boolean类型不能与其他类型进行类型转换
- 不相关的对象类型不能进行转换
- 容量大的类型转换为容量小的类型必须强制进行转换
- 浮点数转换为整数会舍弃小数部分
- 容量小类型可自动转换为容量大的基础类型
- char 类型

　　**byte ，short ，char 进行运算会先类型提升为int**

## 引用

### 反射.class、class.forname() 和 getClass() 的区别

- **相同：**
  - 通过这几种方式，得到的都是java.lang.Class对象；都是类加载的产物
  - 三种情况在生成 Class 对象的时候都会先判断内存中是否已经加载此类。
- **不同：**
  - 出现的时期不同：
    - Class.forname()在运行时加载；
    - Class.class和对象名.getClass()是在编译时加载
  - class.forname() 会装入类并做类的静态初始化
  - Class c = C.class；JVM将使用类C的类装载器将类C加载到内存中(前提是类C还未加载进内存)，不进行类的初始化工作，返回C的Class对象
  - Class c = c.getClass()会对类进行静态初始化、非静态初始化，返回引用运行时真正所指的对象（因为子对象的引用可能会赋给父对象的引用变量中）所属的类的 Class 对象

### 为什么 wait()/notify()方法要放到 Object中呢

　　因为每个对象都可以成为锁监视器对象，所以放到 Object中，可以直接使用。

###
