---
title: Spring 事务
date: 2022-10-30T13:59:09Z
lastmod: 2023-03-18T16:54:19Z
---

# Spring 事务

### 

### 事务的传播性

> 传播性判断处理在 AbstractPlatformTransactionManager的handleExistingTransaction方法实现

#### 事务传播性简介

　　**支持当前事务的情况：**

- **TransactionDefinition.PROPAGATION_REQUIRED：** 如果当前存在事务，则加入该事务；如果当前没有事务，则创建一个新的事务。
- **TransactionDefinition.PROPAGATION_SUPPORTS：** 如果当前存在事务，则加入该事务；如果当前没有事务，则以非事务的方式继续运行。
- **TransactionDefinition.PROPAGATION_MANDATORY：** 如果当前存在事务，则加入该事务；如果当前没有事务，则抛出异常。（mandatory：强制性）

　　**不支持当前事务的情况：**

- **TransactionDefinition.PROPAGATION_REQUIRES_NEW：** 创建一个新的事务，如果当前存在事务，则把当前事务挂起。（如果A方法使用默认的事务传播属性，B方法使用REQUIRES_NEW，此时A方法在内部调用B方法，一旦A方法出现异常，A方法中的事务回滚了，但是B方法并没有回滚，因为A和B方法使用的不是同一个事务，B方法新建了一个事务。但是B方法出现问题抛出异常会导致事务A回滚）
- **TransactionDefinition.PROPAGATION_NOT_SUPPORTED：** 以非事务方式运行，如果当前存在事务，则把当前事务挂起。
- **TransactionDefinition.PROPAGATION_NEVER：** 以非事务方式运行，如果当前存在事务，则抛出异常。

　　**其他情况：**

- **TransactionDefinition.PROPAGATION_NESTED：** 如果当前存在事务，则创建一个事务作为当前事务的嵌套事务来运行；如果当前没有事务，则该取值等价于TransactionDefinition.PROPAGATION_REQUIRED。

> 嵌套事务 ：也就是在进入子事务之前，父事务建立一个回滚点，与当前事务同步提交或回滚。 子事务是父事务的一部分，在父事务还未提交时，子事务一定没有提交。嵌套事务一个非常重要的概念就是内层事务依赖于外层事务。外层事务失败时，会回滚内层事务所做的动作。而内层事务操作失败并不会引起外层事务的回滚。

#### [实现原理](siyuan://blocks/20230318165118-c14byj8)

### 事务失效场景

1. 数据库不支持事务
   1. mysql使用MyISAM引擎就不支持事务
2. 被代理的类没有被Spring管理
3. 被代理的类过早的实例化。
4. 标注事务注解的起点不在抛出异常的范围内
5. 事务的**_起点_**是被this调用，并没有去调用代理类

```java
public void test1(){
    test2();
}
 
@Transactional(rollbackFor = Exception.class)
public void test2(){
   throw new RuntimeException();
}
```

　　正常的事务代理是生成的代理类中调用目标方法的外层有try catch包裹着，如果出现异常会执行回滚操作，但是这里事物的起点是通过this调用的，也就是使用真实的类调用目标方法，目标方法出现异常，也就不会回滚，需要使用注入的方法注入当前对象，然后使用代理类调用目标方法。

6. 方法抛出的异常在方法内捕获，没有被事务拦截器所拦截
7. 抛出的异常与事务能够处理的异常不匹配
8. 未配置事务管理器

### Spring 事务执行过程

> 前置知识 SpringBean 生命周期 ，Spring AOP
