---
title: 浅入浅出 Spring 事务传播实现原理 
date: 2023-03-18T16:51:18Z
lastmod: 2023-03-18T16:51:58Z
---

# 浅入浅出 Spring 事务传播实现原理 

#### 本文目标

* 理解Spring事务管理核心接口
* 理解Spring事务管理的核心逻辑
* 理解事务的传播类型及其实现原理

#### 版本

　　SpringBoot 2.3.3.RELEASE

#### 什么是事务的传播？

　　**Spring 除了封装了事务控制之外，还抽象出了 事务的传播 这个概念，事务的传播并不是关系型数据库所定义的，而是Spring在封装事务时做的增强扩展，**可以通过`@Transactional`​ 指定事务的传播，具体类型如下

|事务传播行为类型|说明|
| ------------------| ------|
|​**`PROPAGATION_REQUIRED`**​|**如果当前没有事务，就新建一个事务，如果已经存在一个事务中，加入到这个事务中。Spring的默认事务传播类型**|
|​**`PROPAGATION_SUPPORTS`**​|**支持当前事务，如果当前没有事务，就以非事务方式执行。**|
|​**`PROPAGATION_MANDATORY`**​|**使用当前的事务，如果当前没有事务，就抛出异常。**|
|​**`PROPAGATION_REQUIRES_NEW`**​|**新建事务，如果当前存在事务，把当前事务挂起（暂停）。**|
|​**`PROPAGATION_NOT_SUPPORTED`**​|**以非事务方式执行操作，如果当前存在事务，就把当前事务挂起。**|
|​**`PROPAGATION_NEVER`**​|**以非事务方式执行，如果当前存在事务，则抛出异常。**|
|​**`PROPAGATION_NESTED`**​|**如果当前存在事务，则在嵌套事务内执行。如果当前没有事务，则执行与PROPAGATION_REQUIRED类似的操作。**|

#### 举个栗子

　　以嵌套事务为例

```java
@Service
public class DemoServiceImpl implements DemoService {
    @Autowired
    private JdbcTemplate jdbcTemplate;

    @Autowired
    private DemoServiceImpl self;

    @Transactional
    @Override
    public void insertDB() {
        String sql = "INSERT INTO sys_user(`id`, `username`) VALUES (?, ?)";
        jdbcTemplate.update(sql, uuid(), "taven");

        try {
            // 内嵌事务将会回滚，而外部事务不会受到影响
            self.nested();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    @Transactional(propagation = Propagation.NESTED)
    @Override
    public void nested() {
        String sql = "INSERT INTO sys_user(`id`, `username`) VALUES (?, ?)";
        jdbcTemplate.update(sql, uuid(), "nested");

        throw new RuntimeException("rollback nested");

    }

    private String uuid() {
        return UUID.randomUUID().toString();
    }
}
JAVA 复制 全屏
```

　　上述代码中，nested()方法标记了事务传播类型为嵌套，如果`nested()`​中抛出异常仅会回滚`nested()`​方法中的sql，不会影响到`insertDB()`​方法中已经执行的sql

> **注意：service 调用内部方法时，如果直接使用this调用，事务不会生效。因此使用this调用相当于跳过了外部的代理类，所以AOP不会生效，无法使用事务**

#### 思考

　　众所周知，**Spring 事务是通过AOP实现的**，**如果是我们自己写一个AOP控制事务，该怎么做呢？**

```tsx
// 伪代码
public Object invokeWithinTransaction() {
    // 开启事务
    connection.beginTransaction();
    try {
        // 反射执行方法
        Object result = invoke();
        // 提交事务
        connection.commit();
        return result;
    } catch(Exception e) {
        // 发生异常时回滚
        connection.rollback();
        throw e;
    } 
  
}
```

　　在这个基础上，**我们来思考一下如果是我们自己做的话，事务的传播该如何实现**

　　以**`PROPAGATION_REQUIRED`**​为例，这个似乎很简单，我们判断一下当前是否有事务（可以考虑使用ThreadLocal存储已存在的事务对象），**如果有事务，那么就不开启新的事务。反之，没有事务，我们就创建新的事务**

　　如果事务是由当前切面开启的，则提交/回滚事务，反之不做处理

　　那么事务传播中描述的挂起（暂停）当前事务，和内嵌事务是如何实现的？

#### 源码入手

　　要阅读事务传播相关的源码，我们先来了解下Spring 事务管理的核心接口与类

1. **TransactionDefinition**  
    该接口定义了事务的所有属性（隔离级别，传播类型，超时时间等等），我们日常开发中经常使用的 `@Transactional`​ 其实最终会被转化为 TransactionDefinition
2. **TransactionStatus**  
    事务的状态，以最常用的实现 DefaultTransactionStatus 为例，该类存储了当前的事务对象，savepoint，当前挂起的事务，是否完成，是否仅回滚等等
3. **TransactionManager**  
    这是一个空接口，直接继承他的 interface 有 **PlatformTransactionManager**（我们平时用的就是这个，默认的实现类DataSourceTransactionManager）以及  
    ReactiveTransactionManager（响应式事务管理器，由于不是本文重点，我们不多说）

　　从上述两个接口来看，**TransactionManager 的主要作用**

* **通过TransactionDefinition开启一个事务，返回TransactionStatus**
* **通过TransactionStatus 提交、回滚事务（实际开启事务的Connection通常存储在TransactionStatus中）**

```java
public interface PlatformTransactionManager extends TransactionManager {
  
    TransactionStatus getTransaction(@Nullable TransactionDefinition definition)
            throws TransactionException;

  
    void commit(TransactionStatus status) throws TransactionException;

  
    void rollback(TransactionStatus status) throws TransactionException;

}

```

4. **TransactionInterceptor事务拦截器，事务AOP的核心类（**支持响应式事务，编程式事务，以及我们常用的标准事务），由于篇幅原因，本文只讨论标准事务的相关实现

　　下面我们**从事务逻辑的入口 TransactionInterceptor 入手**，来看下Spring事务管理的核心逻辑以及事务传播的实现

##### TransactionInterceptor

　　**TransactionInterceptor 实现了MethodInvocation（这是实现AOP的一种方式），**

　　**其核心逻辑在父类TransactionAspectSupport 中**，**方法位置：**​**`TransactionInterceptor::invokeWithinTransaction`**​

```dart
    protected Object invokeWithinTransaction(Method method, @Nullable Class<?> targetClass,
            final InvocationCallback invocation) throws Throwable {
        // If the transaction attribute is null, the method is non-transactional.
        TransactionAttributeSource tas = getTransactionAttributeSource();
        // 当前事务的属性 TransactionAttribute extends TransactionDefinition
        final TransactionAttribute txAttr = (tas != null ? tas.getTransactionAttribute(method, targetClass) : null);
        // 事务属性中可以定义当前使用哪个事务管理器
        // 如果没有定义就去Spring上下文找到一个可用的 TransactionManager
        final TransactionManager tm = determineTransactionManager(txAttr);

        // 省略了响应式事务的处理 ...
      
        PlatformTransactionManager ptm = asPlatformTransactionManager(tm);
        final String joinpointIdentification = methodIdentification(method, targetClass, txAttr);

        if (txAttr == null || !(ptm instanceof CallbackPreferringPlatformTransactionManager)) {
            // Standard transaction demarcation with getTransaction and commit/rollback calls.
            TransactionInfo txInfo = createTransactionIfNecessary(ptm, txAttr, joinpointIdentification);

            Object retVal;
            try {
                // This is an around advice: Invoke the next interceptor in the chain.
                // This will normally result in a target object being invoked.
                // 如果有下一个拦截器则执行，最终会执行到目标方法，也就是我们的业务代码
                retVal = invocation.proceedWithInvocation();
            }
            catch (Throwable ex) {
                // target invocation exception
                // 当捕获到异常时完成当前事务 （提交或者回滚）
                completeTransactionAfterThrowing(txInfo, ex);
                throw ex;
            }
            finally {
                cleanupTransactionInfo(txInfo);
            }

            if (retVal != null && vavrPresent && VavrDelegate.isVavrTry(retVal)) {
                // Set rollback-only in case of Vavr failure matching our rollback rules...
                TransactionStatus status = txInfo.getTransactionStatus();
                if (status != null && txAttr != null) {
                    retVal = VavrDelegate.evaluateTryFailure(retVal, txAttr, status);
                }
            }
            // 根据事务的状态提交或者回滚
            commitTransactionAfterReturning(txInfo);
            return retVal;
        }

        // 省略了编程式事务的处理 ...
    }
```

　　这里代码很多，根据注释的位置，**我们可以把核心逻辑梳理出来**

1. **获取当前事务属性，事务管理器（以注解事务为例，这些都可以通过**​**`@Transactional`**​**来定义）**
2. ​**`createTransactionIfNecessary`**​**，判断是否有必要创建事务**
3. ​**`invocation.proceedWithInvocation`**​**​ 执行拦截器链，最终会执行到目标方法**
4. ​**`completeTransactionAfterThrowing`**​**当抛出异常后，完成这个事务，提交或者回滚，并抛出这个异常**
5. ​**`commitTransactionAfterReturning`**​**​ 从方法命名来看，这个方法会提交事务。但是深入源码中会发现，该方法中也包含回滚逻辑，具体行为会根据当前TransactionStatus的一些状态来决定（也就是说，我们也可以通过设置当前TransactionStatus，来控制事务回滚，并不一定只能通过抛出异常），详见**​**`AbstractPlatformTransact ionManager::commit`**​

　　我们继续，来看看**createTransactionIfNecessary**做了什么

##### TransactionAspectSupport::createTransactionIfNecessary

```kotlin
    protected TransactionInfo createTransactionIfNecessary(@Nullable PlatformTransactionManager tm,
            @Nullable TransactionAttribute txAttr, final String joinpointIdentification) {

        // If no name specified, apply method identification as transaction name.
        if (txAttr != null && txAttr.getName() == null) {
            txAttr = new DelegatingTransactionAttribute(txAttr) {
                @Override
                public String getName() {
                    return joinpointIdentification;
                }
            };
        }

        TransactionStatus status = null;
        if (txAttr != null) {
            if (tm != null) {
                // 通过事务管理器开启事务
                status = tm.getTransaction(txAttr);
            }
            else {
                if (logger.isDebugEnabled()) {
                    logger.debug("Skipping transactional joinpoint [" + joinpointIdentification +
                            "] because no transaction manager has been configured");
                }
            }
        }
      
        return prepareTransactionInfo(tm, txAttr, joinpointIdentification, status);
    }
```

　　**createTransactionIfNecessary中的核心逻辑**

1. **通过PlatformTransactionManager（事务管理器）开启事务**
2. ​**`prepareTransactionInfo`**​**​ 准备事务信息，这个具体做了什么我们稍后再讲**

　　继续来看`PlatformTransactionManager::getTransaction`​，该方法只有一个实现 `AbstractPlatformTransactionManager::getTransaction`​

```java
    public final TransactionStatus getTransaction(@Nullable TransactionDefinition definition)
            throws TransactionException {

        // Use defaults if no transaction definition given.
        TransactionDefinition def = (definition != null ? definition : TransactionDefinition.withDefaults());

        // 获取当前事务，该方法有继承 AbstractPlatformTransactionManager 的子类自行实现
        Object transaction = doGetTransaction();
        boolean debugEnabled = logger.isDebugEnabled();

        // 如果目前存在事务
        if (isExistingTransaction(transaction)) {
            // Existing transaction found -> check propagation behavior to find out how to behave.
            return handleExistingTransaction(def, transaction, debugEnabled);
        }

        // Check definition settings for new transaction.
        if (def.getTimeout() < TransactionDefinition.TIMEOUT_DEFAULT) {
            throw new InvalidTimeoutException("Invalid transaction timeout", def.getTimeout());
        }

        // 传播类型PROPAGATION_MANDATORY, 要求当前必须有事务
        // No existing transaction found -> check propagation behavior to find out how to proceed.
        if (def.getPropagationBehavior() == TransactionDefinition.PROPAGATION_MANDATORY) {
            throw new IllegalTransactionStateException(
                    "No existing transaction found for transaction marked with propagation 'mandatory'");
        }
        // PROPAGATION_REQUIRED, PROPAGATION_REQUIRES_NEW, PROPAGATION_NESTED 不存在事务时创建事务
        else if (def.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRED ||
                def.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW ||
                def.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
            SuspendedResourcesHolder suspendedResources = suspend(null);
            if (debugEnabled) {
                logger.debug("Creating new transaction with name [" + def.getName() + "]: " + def);
            }
            try {
                // 开启事务
                return startTransaction(def, transaction, debugEnabled, suspendedResources);
            }
            catch (RuntimeException | Error ex) {
                resume(null, suspendedResources);
                throw ex;
            }
        }
        else {
            // Create "empty" transaction: no actual transaction, but potentially synchronization.
            if (def.getIsolationLevel() != TransactionDefinition.ISOLATION_DEFAULT && logger.isWarnEnabled()) {
                logger.warn("Custom isolation level specified but no actual transaction initiated; " +
                        "isolation level will effectively be ignored: " + def);
            }
            boolean newSynchronization = (getTransactionSynchronization() == SYNCHRONIZATION_ALWAYS);
            return prepareTransactionStatus(def, null, true, newSynchronization, debugEnabled, null);
        }
    }

```

　　代码很多，重点关注注释部分即可

1. ​**`doGetTransaction`**​**获取当前事务**
2. **如果存在事务，则调用**​**`handleExistingTransaction`**​**处理，这个我们稍后会讲到**

　　**接下来，会根据事务的传播决定是否开启事务**

3. **如果事务传播类型为**​**`PROPAGATION_MANDATORY`**​**，且不存在事务，则抛出异常**
4. **如果传播类型为 ​**​**`PROPAGATION_REQUIRED, PROPAGATION_REQUIRES_NEW, PROPAGATION_NESTED`**​**，且当前不存在事务，则调用**​**`startTransaction`**​**创建事务**
5. **当不满足 3、4时，例如 ​**​**`PROPAGATION_NOT_SUPPORTED`**​**，此时会执行事务同步，但是不会创建真正的事务**

> **Spring 事务同步在之前一篇博客中有讲到，**传送门👉[https://www.jianshu.com/p/7880d9a98a5f](https://www.jianshu.com/p/7880d9a98a5f)

##### Spring 如何管理当前的事务

　　接下来讲讲上面提到的`doGetTransaction`​、`handleExistingTransaction`​，这两个方法是由不同的TransactionManager自行实现的

　　我们以SpringBoot默认的TransactionManager，DataSourceTransactionManager为例

```java
    @Override
    protected Object doGetTransaction() {
        DataSourceTransactionObject txObject = new DataSourceTransactionObject();
        txObject.setSavepointAllowed(isNestedTransactionAllowed());
        ConnectionHolder conHolder =
                (ConnectionHolder) TransactionSynchronizationManager.getResource(obtainDataSource());
        txObject.setConnectionHolder(conHolder, false);
        return txObject;
    }

    @Override
    protected boolean isExistingTransaction(Object transaction) {
        DataSourceTransactionObject txObject = (DataSourceTransactionObject) transaction;
        return (txObject.hasConnectionHolder() && txObject.getConnectionHolder().isTransactionActive());
    }
```

　　结合 `AbstractPlatformTransactionManager::getTransaction`​ 一起来看，`doGetTransaction`​ 其实获取的是当前的Connection。  
判断当前是否存在事务，是判断DataSourceTransactionObject 对象中是否包含connection，以及connection是否开启了事务。

　　我们继续来看下`TransactionSynchronizationManager.getResource(obtainDataSource())`​获取当前connection的逻辑

##### TransactionSynchronizationManager::getResource

```csharp
    private static final ThreadLocal<Map<Object, Object>> resources =
            new NamedThreadLocal<>("Transactional resources");
  
    @Nullable
    // TransactionSynchronizationManager::getResource
    public static Object getResource(Object key) {
        // DataSourceTransactionManager 调用该方法时，以数据源作为key
      
        // TransactionSynchronizationUtils::unwrapResourceIfNecessary 如果key为包装类，则获取被包装的对象
        // 我们可以忽略该逻辑
        Object actualKey = TransactionSynchronizationUtils.unwrapResourceIfNecessary(key);
        Object value = doGetResource(actualKey);
        if (value != null && logger.isTraceEnabled()) {
            logger.trace("Retrieved value [" + value + "] for key [" + actualKey + "] bound to thread [" +
                    Thread.currentThread().getName() + "]");
        }
        return value;
    }

    /**
     * Actually check the value of the resource that is bound for the given key.
     */
    @Nullable
    private static Object doGetResource(Object actualKey) {
        Map<Object, Object> map = resources.get();
        if (map == null) {
            return null;
        }
        Object value = map.get(actualKey);
        // Transparently remove ResourceHolder that was marked as void...
        if (value instanceof ResourceHolder && ((ResourceHolder) value).isVoid()) {
            map.remove(actualKey);
            // Remove entire ThreadLocal if empty...
            if (map.isEmpty()) {
                resources.remove();
            }
            value = null;
        }
        return value;
    }
```

　　看到这里，**我们能明白DataSourceTransactionManager是如何管理线程之间的Connection，ThreadLocal 中存储一个Map，key为数据源对象，value为该数据源在当前线程的Connection**

　　​![](https://upload-images.jianshu.io/upload_images/9949918-966a7caf7ebce006.png)​

　　image.png

　　**DataSourceTransactionManager 在开启事务后，会调用**​**`TransactionSynchronizationManager::bindResource`**​**将指定数据源的Connection绑定到当前线程**

##### AbstractPlatformTransactionManager::handleExistingTransaction

　　我们继续回头看，如果存在事务的情况，如何处理

```tsx
    private TransactionStatus handleExistingTransaction(
            TransactionDefinition definition, Object transaction, boolean debugEnabled)
            throws TransactionException {

        // 如果事务的传播要求以非事务方式执行 抛出异常
        if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NEVER) {
            throw new IllegalTransactionStateException(
                    "Existing transaction found for transaction marked with propagation 'never'");
        }

        // PROPAGATION_NOT_SUPPORTED 如果存在事务，则挂起当前事务，以非事务方式执行
        if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NOT_SUPPORTED) {
            if (debugEnabled) {
                logger.debug("Suspending current transaction");
            }
            // 挂起当前事务
            Object suspendedResources = suspend(transaction);
            boolean newSynchronization = (getTransactionSynchronization() == SYNCHRONIZATION_ALWAYS);
            // 构建一个无事务的TransactionStatus
            return prepareTransactionStatus(
                    definition, null, false, newSynchronization, debugEnabled, suspendedResources);
        }

        // PROPAGATION_REQUIRES_NEW 如果存在事务，则挂起当前事务，新建一个事务
        if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW) {
            if (debugEnabled) {
                logger.debug("Suspending current transaction, creating new transaction with name [" +
                        definition.getName() + "]");
            }
            SuspendedResourcesHolder suspendedResources = suspend(transaction);
            try {
                return startTransaction(definition, transaction, debugEnabled, suspendedResources);
            }
            catch (RuntimeException | Error beginEx) {
                resumeAfterBeginException(transaction, suspendedResources, beginEx);
                throw beginEx;
            }
        }

        // PROPAGATION_NESTED 内嵌事务，就是我们开头举得例子
        if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
            if (!isNestedTransactionAllowed()) {
                throw new NestedTransactionNotSupportedException(
                        "Transaction manager does not allow nested transactions by default - " +
                        "specify 'nestedTransactionAllowed' property with value 'true'");
            }
            if (debugEnabled) {
                logger.debug("Creating nested transaction with name [" + definition.getName() + "]");
            }
            // 非JTA事务管理器都是通过savePoint实现的内嵌事务
            // savePoint：关系型数据库中事务可以创建还原点，并且可以回滚到还原点
            if (useSavepointForNestedTransaction()) {
                // Create savepoint within existing Spring-managed transaction,
                // through the SavepointManager API implemented by TransactionStatus.
                // Usually uses JDBC 3.0 savepoints. Never activates Spring synchronization.
                DefaultTransactionStatus status =
                        prepareTransactionStatus(definition, transaction, false, false, debugEnabled, null);
                // 创建还原点
                status.createAndHoldSavepoint();
                return status;
            }
            else {
                // Nested transaction through nested begin and commit/rollback calls.
                // Usually only for JTA: Spring synchronization might get activated here
                // in case of a pre-existing JTA transaction.
                return startTransaction(definition, transaction, debugEnabled, null);
            }
        }

        // 如果执行到这一步传播类型一定是，PROPAGATION_SUPPORTS 或者 PROPAGATION_REQUIRED
        // Assumably PROPAGATION_SUPPORTS or PROPAGATION_REQUIRED.
        if (debugEnabled) {
            logger.debug("Participating in existing transaction");
        }
      
        // 校验目前方法中的事务定义和已存在的事务定义是否一致
        if (isValidateExistingTransaction()) {
            if (definition.getIsolationLevel() != TransactionDefinition.ISOLATION_DEFAULT) {
                Integer currentIsolationLevel = TransactionSynchronizationManager.getCurrentTransactionIsolationLevel();
                if (currentIsolationLevel == null || currentIsolationLevel != definition.getIsolationLevel()) {
                    Constants isoConstants = DefaultTransactionDefinition.constants;
                    throw new IllegalTransactionStateException("Participating transaction with definition [" +
                            definition + "] specifies isolation level which is incompatible with existing transaction: " +
                            (currentIsolationLevel != null ?
                                    isoConstants.toCode(currentIsolationLevel, DefaultTransactionDefinition.PREFIX_ISOLATION) :
                                    "(unknown)"));
                }
            }
            if (!definition.isReadOnly()) {
                if (TransactionSynchronizationManager.isCurrentTransactionReadOnly()) {
                    throw new IllegalTransactionStateException("Participating transaction with definition [" +
                            definition + "] is not marked as read-only but existing transaction is");
                }
            }
        }
        boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
        // 构建一个TransactionStatus，但不开启事务
        return prepareTransactionStatus(definition, transaction, false, newSynchronization, debugEnabled, null);
    }
```

　　这里代码很多，逻辑看上述注释即可。这里终于看到了期待已久的挂起事务和内嵌事务了，我们还是看一下DataSourceTransactionManager的实现

* 挂起事务：通过`TransactionSynchronizationManager::unbindResource`​ 根据数据源获取当前的Connection，并在resource中移除该Connection。之后会将该Connection存储到TransactionStatus对象中

```tsx
    // DataSourceTransactionManager::doSuspend
    @Override
    protected Object doSuspend(Object transaction) {
        DataSourceTransactionObject txObject = (DataSourceTransactionObject) transaction;
        txObject.setConnectionHolder(null);
        return TransactionSynchronizationManager.unbindResource(obtainDataSource());
    }
```

　　在事务提交或者回滚后，调用 `AbstractPlatformTransactionManager::cleanupAfterCompletion`​会将TransactionStatus 中缓存的Connection重新绑定到resource中

* 内嵌事务：通过关系型数据库的savePoint实现，提交或回滚的时候会判断如果当前事务为savePoint则释放savePoint或者回滚到savePoint，具体逻辑参考`AbstractPlatformTransactionManager::processRollback`​ 和 `AbstractPlatformTransactionManager::processCommit`​

　　至此，事务的传播源码分析结束

#### **prepareTransactionInfo**

　　上文留下了一个问题，prepareTransactionInfo 方法做了什么，我们先来看下`TransactionInfo`​的结构

```java
    protected static final class TransactionInfo {

        @Nullable
        private final PlatformTransactionManager transactionManager;

        @Nullable
        private final TransactionAttribute transactionAttribute;

        private final String joinpointIdentification;

        @Nullable
        private TransactionStatus transactionStatus;

        @Nullable
        private TransactionInfo oldTransactionInfo;
      
        // ...
    }
```

　　该类在Spring中的作用，**是为了内部传递对象。ThreadLocal中存储了最新的TransactionInfo，通过当前TransactionInfo可以找到他的oldTransactionInfo。每次创建事务时会新建一个TransactionInfo（无论有没有真正的事务被创建）存储到ThreadLocal中，在每次事务结束后，会将当前ThreadLocal中的TransactionInfo重置为oldTransactionInfo，这样的结构形成了一个链表，使得Spring事务在逻辑上可以无限嵌套下去**
