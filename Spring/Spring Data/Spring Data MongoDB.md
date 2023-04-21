---
title: Spring Data MongoDB
date: 2022-10-30T14:42:14Z
lastmod: 2022-10-30T14:42:14Z
---

# Spring Data MongoDB

### 映射注释

　　`@Id`：用于标记 id 字段，没有标记此字段的实体也会自动生成 id 字段，但是我们无法通过实体来获取 id。id 建议使用 ObjectId 类型来创建。

　　`@Document`：用于标记此实体类是 mongodb 集合映射类。可以使用 collection 参数指定集合名称。特别需要注意的是如果实体类没有为任何字段创建索引将不会自动创建集合。

　　`@Indexed`：用于标记为某一字段创建索引。direction 参数可以指定排序方向，升或降序。

　　`@CompoundIndex`：用于创建复合索引。def 参数可以定义复合索引的字段及排序方向。

　　`@Transient`：被该注解标注的，将不会被录入到数据库中。只作为普通的 javaBean 属性。

　　`@PersistenceConstructor`：用于声明构造函数，作用是把从数据库取出的数据实例化为对象。

　　`@Field`：用于指定某一个字段映射到数据库中的名称。

　　`@DBRef`：用于指定与其他集合的级联关系，但是需要注意的是并不会自动创建级联集合。

### 配置

```xml
<dependency>
   <groupId>org.springframework.boot</groupId>
   <artifactId>spring-boot-starter-data-mongodb</artifactId>
</dependency>
```

```yml
spring:
  application:
    name: online-manage-cms
  data:
    mongodb:
      uri: "mongodb://root:root@localhost:27017/"
      database: online_edu
```

### 数据操作

#### MongoDBRepository

1. 按照规定定义接口名称不需要实现
2. 要有主键值才可以用 save(有就更新，没有就插入)。所以就算没有 ID 也增加这个字段是好的。（id 要是 String 才会自动为你生面 ID 号（保存时可以没有值），int 要你自己做惟一处理和输入值）
3. DATE 不能作为主键使用。

　　Repository 接口是 Spring Data 的一个核心接口，它不提供任何方法，开发者需要在自己定义的接口中声明需要的方法

```java
public interface Repository<T, ID extends Serializable> { }
```

　　Spring Data 可以让我们只定义接口，只要遵循 Spring Data 的规范，就无需写实现类。  
与继承 Repository 等价的一种方式，就是在持久层接口上使用 [@RepositoryDefinition ]() 注解，并为其指定 domainClass 和 idClass 属性,两种方式没有区别。

　　Repository 提供了最基本的数据访问功能，其几个子接口则扩展了一些功能。它们的继承关系如下：

```java
Repository： 仅仅是一个标识，表明任何继承它的均为仓库接口类
CrudRepository： 继承 Repository，实现了一组 CRUD 相关的方法 
PagingAndSortingRepository： 继承 CrudRepository，实现了一组分页排序相关的方法 
MongoRepository： 继承 PagingAndSortingRepository，实现一组 mongodb规范相关的方法
```

　　自定义的 XxxxRepository 需要继承 MongoRepository，这样的 XxxxRepository 接口就具备了通用的数据访问控制层的能力(CURD 的操作功能)。

#### MongoDBTemplate

　　MongoTemplate 是数据库和代码之间的接口，对数据库的操作都在它里面。

```java
MongoTemplate 实现了 interface MongoOperations。
MongoDB documents 和 domain classes` 之间的映射关系是通过实现了MongoConverter这个interface的类来实现的。
MongoTemplate 提供了非常多的操作MongoDB的方法。 它是线程安全的，可以在多线程的情况下使用。
MongoTemplate 实现了 MongoOperations 接口, 此接口定义了众多的操作方法如"find", "findAndModify", "findOne", "insert", "remove", "save", "update" and "updateMulti"等。
MongoTemplate转换domain object为DBObject,缺省转换类为MongoMappingConverter,并提供了Query, Criteria, and Update等流式API。
```

##### 增

```
mongoTemplate.insert(Collection<? extends Object> batchToSave, Class<?> entityClass) return void
mongoTemplate.insert(Collection<? extends Object> batchToSave, String collectionName) return void
mongoTemplate.insert(Object objectToSave) return void
mongoTemplate.insert(Object objectToSave, String collectionName) return void
mongoTemplate.insertAll(Collection<? extends Object> objectsToSave) return void
mongoTemplate.save(Object objectToSave) return void
mongoTemplate.save(Object objectToSave, String collectionName) return void
//在插入之前，判断数据是否存在，如果不存在则插入；如果存在则更新；多条满足时,只会修改一条数据
mongoTemplate.upsert(Query query, Update update, String collectionName)  return UpdateResult
```

##### 删

```
mongoTemplate.findAndRemove(Query query, Class<T> entityClass)  return <T>
mongoTemplate.findAndRemove(Query query, Class<T> entityClass, String collectionName) return <T>
mongoTemplate.remove(Object object) return com.mongodb.client.result.DeleteResult
mongoTemplate.remove(Object object, String collectionName) return com.mongodb.client.result.DeleteResult
mongoTemplate.remove(Query query, Class<?> entityClass) return com.mongodb.client.result.DeleteResult
mongoTemplate.remove(Query query, Class<?> entityClass, String collectionName) return com.mongodb.client.result.DeleteResult
mongoTemplate.remove(Query query, String collectionName) return com.mongodb.client.result.DeleteResult
```

##### 改

```
mongoTemplate.findAndModify(Query query, Update update, Class<T> entityClass)  return <T>
mongoTemplate.findAndModify(Query query, Update update, Class<T> entityClass, String collectionName)  return <T>
mongoTemplate.findAndModify(Query query, Update update, FindAndModifyOptions options, Class<T> entityClass)  return <T>
mongoTemplate.findAndModify(Query query, Update update, FindAndModifyOptions options, Class<T> entityClass, String collectionName)  return <T>
```

##### 查询

```
mongoTemplate.find(Query query, Class<T> entityClass)  return List<T>
mongoTemplate.find(Query query, Class<T> entityClass, String collectionName) return List<T>
mongoTemplate.findAll(Class<T> entityClass)  return List<T>
mongoTemplate.findAll(Class<T> entityClass, String collectionName)  return List<T>
mongoTemplate.findAllAndRemove(Query query, Class<T> entityClass)  return List<T>
mongoTemplate.findAllAndRemove(Query query, Class<T> entityClass, String collectionName)  return List<T>
mongoTemplate.findAllAndRemove(Query query, String collectionName)  return List<T>
mongoTemplate.findById(Object id, Class<T> entityClass) return <T>
mongoTemplate.findById(Object id, Class<T> entityClass, String collectionName) return <T>
mongoTemplate.findOne(Query query, Class<T> entityClass) return <T>
mongoTemplate.findOne(Query query, Class<T> entityClass, String collectionName) return <T>
mongoTemplate.count(Query query) 查询总数
```

#### MongoTemplate 核心操作类：Criteria 和 Query

```java
Criteria 类：封装所有的语句，以方法的形式查询。
Query 类：将语句进行封装或者添加排序之类的操作。
```

##### 根据字段进行查询

```java
Query query = new Query(Criteria.where("user").is("xxx"));  //{ "user" : "xxx" }, Fields: { }, Sort: { }
```

##### and 多条件查询

```java
Query query = new Query(Criteria.where("user").is("xxx").and("age").is(xx)); //{ "user" : "xxx", "age" : xxx }, Fields: { }, Sort: { }
```

##### or 条件查询

```java
Query query = new Query(Criteria.where("user").is("xxx")
            .orOperator(Criteria.where("age").is(xx), Criteria.where("sign").exists(true))); //{"user": "xx", $or: [{ "age": xx}, { "sign": {$exists: true}}]})
Query query = new Query(new Criteria().orOperator(Criteria.where("age").is(18), Criteria.where("sign").exists(true))); //{ "$or" : [{ "age" : 18 }, { "sign" : { "$exists" : true } }] }, Fields: { }, Sort: { }
```

##### in 条件查询

```java
 Query query = new Query(Criteria.where("age").in(Arrays.asList(18, 20, 30))); //{ "age" : { "$in" : [18, 20, 30] } }, Fields: { }, Sort: { }
```

##### 数值比较

　　数值的比较大小，主要使用的是 get, gt, lt, let

```java
    // age > 18
    Query query = new Query(Criteria.where("age").gt(18));//{ "age" : { "$gt" : 18 } }, Fields: { }, Sort: { }
    // age >= 18
    query = new Query(Criteria.where("age").gte(18));//{ "age" : { "$gte" : 18 } }, Fields: { }, Sort: { }
    // age < 20
    Query query = new Query(Criteria.where("age").lt(20));//{ "age" : { "$lt" : 20 } }, Fields: { }, Sort: { }
    // age <= 20
    query = new Query(Criteria.where("age").lte(20));//{ "age" : { "$lte" : 20 } }, Fields: { }, Sort: { }
```

##### 正则查询

```java
Query query = new Query(Criteria.where("user").regex("^xxx")); //{ "user" : { "$regex" : "^xxx", "$options" : "" } }, Fields: { }, Sort: { }
```

##### 排序

```java
//sort查询用with衔接
Query query = Query.query(Criteria.where("user").is("xxx")).with(Sort.by("age")); //{ "user" : "xxx" }, Fields: { }, Sort: { "age" : 1 }
```

##### 分页

```java
    // limit限定查询2条
    Query query = Query.query(Criteria.where("user").is("一灰灰blog")).with(Sort.by("age")).limit(2);//{ "user" : "xx" }, Fields: { }, Sort: { "age" : 1 }
    // skip()方法来跳过指定数量的数据
    query = Query.query(Criteria.where("user").is("一灰灰blog")).with(Sort.by("age")).skip(2);//{ "user" : "xxx" }, Fields: { }, Sort: { "age" : 1 }
```

#### MongoTemplate 聚合函数查询 Aggregation

```java
<O> AggregationResults<O> aggregate(Aggregation aggregation, Class<?> inputType, Class<O> outputType)

<O> AggregationResults<O> aggregate(Aggregation aggregation, String collectionName, Class<O> outputType)

<O> AggregationResults<O> aggregate(TypedAggregation<?> aggregation, Class<O> outputType)

<O> AggregationResults<O> aggregate(TypedAggregation<?> aggregation, String inputCollectionName, Class<O> outputType)

<T> ExecutableAggregationOperation.ExecutableAggregation<T> aggregateAndReturn(Class<T> domainType)

<O> CloseableIterator<O> aggregateStream(Aggregation aggregation, Class<?> inputType, Class<O> outputType)

<O> CloseableIterator<O> aggregateStream(Aggregation aggregation, String collectionName, Class<O> outputType)

<O> CloseableIterator<O> aggregateStream(TypedAggregation<?> aggregation, Class<O> outputType)

<O> CloseableIterator<O> aggregateStream(TypedAggregation<?> aggregation, String inputCollectionName, Class<O> outputType)
```

##### 分组 group

```java
    // 根据用户名进行分组统计，每个用户名对应的数量
    // aggregate([ { "$group" : { "_id" : "user" , "userCount" : { "$sum" : 1}}}] )
    Aggregation aggregation = Aggregation.newAggregation(Aggregation.group("user").count().as("userCount"));
    AggregationResults<Map> ans = mongoTemplate.aggregate(aggregation, COLLECTION_NAME, Map.class);
```
