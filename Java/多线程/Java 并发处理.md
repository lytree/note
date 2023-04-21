---
title: Java 并发处理 
date: 2022-10-30T13:51:31Z
lastmod: 2022-10-30T13:54:38Z
---

# Java 并发处理 

---

## 多线程特性

- 原子性
  - 一个或多个操作要么全部执行并且执行过程中不会被任何因素打断，要么就不执行。
- 可见性
  - 当多个线程同时访问一个变量时，一个线程修改了这个变量，其他线程能够立即看见修改的值。单线程不存在可见性问题。
- 有序型
  - 程序的执行按照代码的先后顺序执行

## 对象的发布和逸出

### 发布

> "发布( Publish)"一个对象的意思是指,使对象能够在当前作用域之外的代码中使用

1. 一个指向该对象的引用保存到其他代码可以访问的地方

```java
public class Person(){
    private String name;
    private int age;
}

public class Record{
    Person p;
}

```

2. 在某一个非私有的方法中返回该引用

```java
public class Person(){
    private String name;
    private int age;

    public Person get(){
        return new Person();
    }
}
```

3. 将引用传递到其他类的方法中

```java
public class Person(){
    private String name;
    private int age;
}

public class Record{
    private void getPersonMessage(Person p){
        .....
    }
}
```

### 逸出

> 当某个不应该发布的对象被发布时,这种情况就被称为逸出( Escape).

1. 内部的可变状态逸出

```java
class UnsafeStates{
    private String[] states = {"AK","AL",...};
    public String[] getStates(){
        return states;
    }
}
//数组states本事私有的变量，但是以public公有的方式发布出去，导致states已经逸出了它所在的作用域，任何调用者都能修改这个数组的内容
```

2. 隐式地使用this引用导致逸出

```java
public class ThisEscape{
    public ThisEscape(EventSource source){
        source.registerListener(
            new EventListener(){
                public void onEvent(Event e){
                    dosomething(e);
                }
            })
        //在这里count初始化为1
        count = 1;
    }
}
//当ThisEscape发布EventListener时，也隐含地发布了ThisEscape实例本身。因为在这个内部类的实例中包含了对ThisEscape实例的隐含引用。
//this逸出会导致ThisEscape也发布出去，也就是ThisEscape还没有构建完成就发布出去，也就是count=1;这一句还没执行就发布了ThisEscape对象，如果要使用count时,很有可能会出现对象不一致的状态
```

　　**使用工厂方法来防止this引用在构造函数过程中逸出**

```java
public class SafeListener{
    private final EventListener listener;
    private SafeListener(){
        listener = new EventListener(){
            public void onEvent(Event e){
                dosomething(e);
            }
        };
    }
    public static SafeListener newInstance(EventSource source){
        SafeListener safe = new SafeListener();
        source.registerListener(safe.listener);
        return safe;
    }
}
//保证在对象为构造完成之前，是不会发布该对象
```

## 线程封闭

1. Ad-hoc线程封闭。

> 维护线程封闭性的职责完全由程序实现承担，可用性不高。

　　Ad-hoc 线程封闭下的一个特例适用于 volatile 变量。 只要确保 volatile 变量仅从单个线程写入，就可以安全地对共享 volatile 变量执读 - 改 - 写操作。

2. 栈封闭。

　　局部变量，无并发问题，在项目中使用最多，简单说就是局部变量，方法的变量都拷贝到线程的堆栈中，只有这个线程能访问到。尽量少使用全局变量（变量不是常量）

3. ThreadLocal类。

## ThreadLocal

　　ThreadLocal 提供线程局部变量，即为使用相同变量的每一个线程维护一个该变量的副本。当某些数据是以线程为作用域并且不同线程具有不同的数据副本的时候，就可以考虑采用 ThreadLocal，比如数据库连接 Connection，每个请求处理线程都需要，但又不相互影响，就是用 ThreadLocal 实现。

### 为何要Entry使用用弱引用

　　如果强引用，即使ThreadLocal对象为null ，ThreadLocalMap的key依旧会指向ThreadLocal对象，会造成内存泄漏。使用弱引用依旧会造成内存泄漏，如果key释放，value没有释放依旧会造成内存泄漏，在不需要时进行ThreadLocal.remove操作。

　　分析

　　![1-9.png](assets/net-img-1582875452927-44c67a88-5cee-48b1-a764-a26d40a77538-20221030135540-f7dfpln.png)

- 在 ThreadLocal 类中定义了一个 ThreadLocalMap，
- 每一个 Thread 都有一个 ThreadLocalMap 类型的变量 threadLocals
- threadLocals 内部有一个 Entry，Entry 的 key 是 ThreadLocal 对象实例，value 就是共享变量副本
- ThreadLocal 的 get 方法就是根据 ThreadLocal 对象实例获取共享变量副本
- ThreadLocal 的 set 方法就是根据 ThreadLocal 对象实例保存共享变量副本

## 原子类

　　Java 的 java.util.concurrent.atomic 包里面提供了很多可以进行原子操作的类，分为以下四类：

- 原子更新基本类型：AtomicInteger、AtomicBoolean、AtomicLong
- 原子更新数组：AtomicIntegerArray、AtomicLongArray
- 原子更新引用：AtomicReference、AtomicStampedReference 等
- 原子更新属性：AtomicIntegerFieldUpdater、AtomicLongFieldUpdater

　　提供这些原子类的目的就是为了解决基本类型操作的非原子性导致在多线程并发情况下引发的问题。

### 非原子问题演示

　　i++并不是原子操作

```java
import java.util.concurrent.atomic.AtomicInteger;

public class AtomicClass {
    static int n = 0;
    public static void main(String[] args) throws InterruptedException {
        int j = 0;
        while(j<100){
            n = 0;
            Thread t1 = new Thread(){
                public void run(){
                    for(int i=0; i<1000; i++){
                        n++;
                    }
                }
            };
            Thread t2 = new Thread(){
                public void run(){
                    for(int i=0; i<1000; i++){
                        n++;
                    }
                }
            };
            t1.start();
            t2.start();
            t1.join();
            t2.join();
            System.out.println("n的最终值是："+n);
            j++;
        }

    }
}
```

　　结果不一定全是 2000

### 非原子问题的原子解决

```java
import java.util.concurrent.atomic.AtomicInteger;

public class AtomicClass {
    static AtomicInteger n;
    public static void main(String[] args) throws InterruptedException {
        int j = 0;
        while(j<100){
            n = new AtomicInteger(0);
            Thread t1 = new Thread(){
                public void run(){
                    for(int i=0; i<1000; i++){
                        n.getAndIncrement();
                    }
                }
            };
            Thread t2 = new Thread(){
                public void run(){
                    for(int i=0; i<1000; i++){
                        n.getAndIncrement();
                    }
                }
            };
            t1.start();
            t2.start();
            t1.join();
            t2.join();
            System.out.println("n的最终值是："+n);
            j++;
        }

    }
}
```

　　**原理**

　　![1-10.png](assets/net-img-1582875462202-61de3608-d606-44f6-b8d0-c82da3a7acd8-20221030135541-a1152vd.png)

### CAS 的 ABA 问题

　　当前内存的值一开始是 A，被另外一个线程先改为 B 然后再改为 A，那么当前线程访问的时候发现是 A，则认为它没有被其他线程访问过。在某些场景下这样是存在错误风险的。如下图：

　　![1-11.png](assets/net-img-1582875470139-ea71ffbe-3cbd-44d8-8560-1e72b3a4bab4-20221030135541-hkuw6hs.png)

### AtomicStampedReference 解决 ABA 问题

```
AtomicStampedReference(初始值，时间戳)：构造函数设置初始值和时间戳
getStamp：获取时间戳
getReference：获取预期值
compareAndSet(预期值，更新值，预期时间戳，更新时间戳)：实现CAS时间戳和预期值对比
```

```java
import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.atomic.AtomicStampedReference;

public class AtomicClass {
    static AtomicStampedReference<Integer> n;
    public static void main(String[] args) throws InterruptedException {
        int j = 0;
        while(j<100){
            n = new AtomicStampedReference<Integer>(0,0);
            Thread t1 = new Thread(){
                public void run(){
                    for(int i=0; i<1000; i++){
                        int stamp;
                        Integer reference;
                        do{
                            stamp = n.getStamp();
                            reference = n.getReference();
                        } while(!n.compareAndSet(reference, reference+1, stamp, stamp+1));
                    }
                }
            };
            Thread t2 = new Thread(){
                public void run(){
                    for(int i=0; i<1000; i++){
                        int stamp;
                        Integer reference;
                        do{
                            stamp = n.getStamp();
                            reference = n.getReference();

                        } while(!n.compareAndSet(reference, reference+1, stamp, stamp+1));
                    }
                }
            };
            t1.start();
            t2.start();
            t1.join();
            t2.join();
            System.out.println("n的最终值是："+n.getReference());
            j++;
        }

    }
}
```

　　**注意：采用 AtomicStampedReference 会降低性能，慎用。**

## Volatile 关键字

### 非原子性的64位操作

　　非Volatile修饰的64位变量（double和long类型），JVM允许将64位的读操作和写操作分为两个32位的操作，如果这两个操作在不同线程执行，可能就会导致数据丢失。在多线程中使用共享且可变的long和double类型变量，必须使用Volatile修饰或加锁，否则会有线程安全问题。

### 原理

　　Java把处理器多级抽象化为JMM，及线程私有化的工作内存和线程共有的主内存，每个线程从主内村拷贝所需数据到自己的工作内存处理，在重新写回主内存。volatile原理就是当线程修改volatile修饰的变量时，要立即写入内存，当线程读取被volatile修饰的变量时，要立即到主存中读取，保证可见性。

### 作用

> 一个共享变量（类的成员变量、类的静态成员变量）被 volatile 修饰之后，那么就具备了两层语义：

- 保证了不同线程对这个变量进行操作时的可见性，即一个线程修改了某个变量的值，这新值对其他线程来说是立即可见的。（注意：不保证原子性）
- 禁止进行指令重排序。（保证变量所在行的有序性）
  - 当程序执行到 volatile 变量的读操作或者写操作时，在其前面的操作的更改肯定全部已经进行，且结果已经对后面的操作可见；在其后面的操作肯定还没有进行；
  - 在进行指令优化时，不能将在对 volatile 变量访问的语句放在其后面执行，也不能把 volatile 变量后面的语句放到其前面执行

### 内存屏障

　　**MESI CPU缓存一致性协议**
屏障两边的指令不可重排
![image.png](assets/net-img-1615619423077-dea312ee-787e-4a28-b723-199f76a214f7-20221030135542-mx20k3e.png)
![image.png](assets/net-img-1615619453470-bdab6e23-1385-4538-9c91-ceefcd05aa27-20221030135542-vyoxljv.png)
**应用场景**

　　基于 volatile 的作用，使用 volatile 必须满足以下两个条件：

- 对变量的写操作不依赖于当前值
- 该变量没有包含在具有其他变量的不变式中
- 在访问变量时不需要加锁
  **常见应用场景如下：**
  状态量标记：

```java
volatile boolean flag = false;
 
while(!flag){
    doSomething();
}
 
public void setFlag() {
    flag = true;
}


volatile boolean inited = false;
//线程1:
context = loadContext(); 
inited = true;           
 
//线程2:
while(!inited ){
sleep()
}
doSomethingwithconfig(context);
```

　　双重校验：

```java
class Singleton{
    private volatile static Singleton instance = null;
 
    private Singleton() {
 
    }
 
    public static Singleton getInstance() {
        if(instance==null) {
            synchronized (Singleton.class) {
                if(instance==null)
                    instance = new Singleton();
            }
        }
        return instance;
    }
}
```

### 局限

　　Volatile修饰变量只能保证可见性，不能保证原子性

##
