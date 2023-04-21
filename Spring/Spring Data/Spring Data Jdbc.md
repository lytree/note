---
title: Spring Data Jdbc
date: 2022-10-30T14:42:56Z
lastmod: 2022-10-30T14:42:56Z
---

# Spring Data Jdbc

## SpringDataJdbc结合Mybatis配置

```java
/**
 * @author PrideYang
 */
@Configuration(proxyBeanMethods = false)
public class MyBatisJdbcConfiguration extends AbstractJdbcConfiguration {


  private @Autowired
  SqlSession session;
  @Autowired
  private SnowFlake snowFlake;

  /**
   * 自定义mybatis命名空间
   *
   * @param operations
   * @param jdbcConverter
   * @param context
   * @param dialect
   * @return
   */
  @Bean
  @Override
  public DataAccessStrategy dataAccessStrategyBean(NamedParameterJdbcOperations operations, JdbcConverter jdbcConverter,
      JdbcMappingContext context, Dialect dialect) {

    return MyBatisDataAccessStrategy.createCombinedAccessStrategy(context, jdbcConverter, operations, session, new MybatisNamespaceStrategy(), dialect);
  }

  /**
   * 保存之前处理id或更新创建时间
   *
   * @return
   */
  @Bean
  BeforeSaveCallback<BaseIdEntity> beforeSaveCallback() {

    return (baseIdEntity, mutableAggregateChange) -> {
      if (baseIdEntity.getCreateTime() == null) {
        baseIdEntity.setCreateTime(LocalDateTime.now());
      }
      baseIdEntity.setUpdateTime(LocalDateTime.now());
      if (baseIdEntity.getId() == null) {
        baseIdEntity.setId(snowFlake.nextId());
      }
      return baseIdEntity;
    };
  }

}

```

```java
/**
 * @author PrideYang
 */
@Configuration
public class MybatisNamespaceStrategy implements NamespaceStrategy {

  @Override
  public String getNamespace(Class<?> domainType) {
    return "top.yang.mapper.".concat(domainType.getSimpleName()).concat("Mapper");
  }
}

```
