---
title: Spring Boot 扩展点
date: 2022-10-30T14:37:33Z
lastmod: 2022-10-30T14:39:42Z
---

# Spring Boot 扩展点

## FactoryBean

> BeanFactory是Bean的工厂，可以帮我们生成想要的Bean，而FactoryBean就是一种Bean的类型。当往容器中注入class类型为FactoryBean的类型的时候，最终生成的Bean是用过FactoryBean的getObject获取的。

- 在Mybatis中的使用

　　Mybatis在整合Spring的时候，就是通过FactoryBean来实现的，这也就是为什么在Spring的Bean中可以注入Mybatis的Mapper接口的动态代理对象的原因。

```java
public class MapperFactoryBean<T> extends SqlSessionDaoSupport implements FactoryBean<T> {
  
  // mapper的接口类型
  private Class<T> mapperInterface;
 
  @Override
  public T getObject() throws Exception {
    // 通过SqlSession获取接口的动态搭理对象
    return getSqlSession().getMapper(this.mapperInterface);
  }
  
  @Override
  public Class<T> getObjectType() {
    return this.mapperInterface;
  }
  
}
```

1. getObject方法的实现就是返回通过SqlSession获取到的Mapper接口的动态代理对象。
2. @MapperScan注解的作用就是将每个接口对应的MapperFactoryBean注册到Spring容器的

- 在OpenFeign中的使用

　　FeignClient接口的动态代理也是通过FactoryBean注入到Spring中的。

```java
class FeignClientFactoryBean implements FactoryBean<Object>, InitializingBean, ApplicationContextAware {
    
    // FeignClient接口类型
    private Class<?> type;
    
    @Override
   public Object getObject() throws Exception {
      return getTarget();
   }
    
    @Override
   public Class<?> getObjectType() {
      return type;
   }
}
```

1. getObject方法是调用getTarget方法来返回的动态代理。
2. @EnableFeignClients注解的作用就是将每个接口对应的FeignClientFactoryBean注入到Spring容器的。

> **一般来说，FactoryBean 比较适合那种复杂Bean的构建，在其他框架整合Spring的时候用的比较多。**

## `@Import注解`

> 在很多情况下，@EnbaleXXX这种格式的注解，都是通过@Import注解起作用的，代表开启了某个功能。
> @Import的核心作用就是导入配置类，并且还可以根据配合（比如@EnableXXX）使用的注解的属性来决定应该往Spring中注入什么样的Bean。

1. 配置类实现了 ImportSelector 接口

```java
public interface ImportSelector {


	String[] selectImports(AnnotationMetadata importingClassMetadata);

	@Nullable
	default Predicate<String> getExclusionFilter() {
		return null;
	}

}
```

　　当配置类实现了ImportSelector 接口的时候，就会调用`selectImports`方法的实现，获取一批类的全限定名，将这些Bean注册到容器中去

2. 配置类实现了 ImportBeanDefinitionRegstrar 接口

```java
public interface ImportBeanDefinitionRegistrar {

   default void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry,BeanNameGenerator importBeanNameGenerator) {
       registerBeanDefinitions(importingClassMetadata, registry);
   }

   default void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
   }

}
```

　　当配置类实现了 ImportBeanDefinitionRegistrar 接口，你就可以自定义往容器中注册想注入的Bean。这个接口相比与 ImportSelector 接口的主要区别就是，ImportSelector接口是返回一个类，你不能对这个类进行任何操作，但是 ImportBeanDefinitionRegistrar 是可以自己注入 BeanDefinition，可以添加属性之类的。

3. 普通的配置类（使用Configuration注释的类）

## Bean生命周期

　　Bean创建和销毁过程中调用的顺序
![](https://cdn.nlark.com/yuque/0/2022/webp/381674/1665817884174-d826c87b-6793-40b9-a020-543434123781.webp#clientId=uf33e3d02-cc27-4&crop=0&crop=0&crop=1&crop=1&errorMessage=unknown%20error&from=paste&id=u23a9599c&margin=%5Bobject%20Object%5D&originHeight=910&originWidth=992&originalType=url&ratio=1&rotation=0&showTitle=false&status=error&style=none&taskId=u1c9c3773-95ce-40a7-8e98-388bbf8a354&title=)
几个Aware接口以及它们的作用

|**接口**|**作用**|
| ------------------------------| ---------------------------------------|
|ApplicationContextAware|注入ApplicationContext|
|ApplicationEventPublisherAware|注入ApplicationEventPublisher事件发布器|
|BeanFactoryAware|注入BeanFactory|
|BeanNameAware|注入Bean的名称|

## BeanPostProcessor

> Bean的后置处理器，在Bean创建的过程中起作用。

　　**Spring内置的BeanPostProcessor**

|BeanPostProcessor|作用|
| --------------------------------------| ----------------------------------------------|
|AutowiredAnnotationBeanPostProcessor|处理@Autowired、@Value注解|
|CommonAnnotationBeanPostProcessor|处理@Resource、@PostConstruct、@PreDestroy注解|
|AnnotationAwareAspectJAutoProxyCreator|处理一些注解或者是AOP切面的动态代理|
|ApplicationContextAwareProcessor|处理Aware接口注入的|
|AsyncAnnotationBeanPostProcessor|处理@Async注解|
|ScheduledAnnotationBeanPostProcessor|处理@Scheduled注解|

　　**BeanPostProcessor在Dubbo中的使用**

```java
public class ReferenceAnnotationBeanPostProcessor 
       extends AbstractAnnotationBeanPostProcessor 
       implements ApplicationContextAware, BeanFactoryPostProcessor {
        // 忽略
}
```

## BeanFactoryPostProcessor

> 对BeanFactory，也就是Spring容器的处理
> **BeanFactoryPostProcessor是可以对Spring容器做处理的，方法的入参就是Spring的容器，通过这个接口，就对容器进行为所欲为的操作。**

```java
public interface BeanFactoryPostProcessor {

	void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;

}

```

## Spring SPI机制 (**SpringFactoriesLoader**)

> Spring的SPI机制规定，配置文件必须在classpath路径下的META-INF文件夹内，文件名必须为spring.factories，文件内容为键值对，一个键可以有多个值，只需要用逗号分割就行，同时键值都需要是类的全限定名。但是键和值可以没有任何关系，当然想有也可以有。

### 自动装配(AutoConfigurationImportSelector)

### ProperySourceLoader

```java
public interface PropertySourceLoader {

   //可以支持哪种文件格式的解析
   String[] getFileExtensions();

   // 解析配置文件，读出内容，封装成一个PropertySource<?>结合返回回去
   List<PropertySource<?>> load(String name, Resource resource) throws IOException;

}
```

#### PropertiesPropertySourceLoader

> 可以解析properties或者xml结尾的配置文件

#### YamlPropertySourceLoader

> 解析以yml或者yaml结尾的配置文件

　　**SpringBoot会先通过SPI机制加载所有PropertySourceLoader，然后遍历每个PropertySourceLoader，判断当前遍历的PropertySourceLoader，通过getFileExtensions获取到当前PropertySourceLoader能够支持哪些配置文件格式的解析，让后跟当前需要解析的文件格式进行匹配，如果能匹配上，那么就会使用当前遍历的PropertySourceLoader来解析配置文件。**

### ApplicationContextInitializer

> ApplicationContextInitializer也是SpringBoot启动过程的一个扩展点。

　　SPI加载ApplicationContextInitializer
然后遍历所有的实现，依次调用
