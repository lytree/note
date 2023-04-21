---
title: Spring Boot 源码
date: 2022-10-30T14:39:20Z
lastmod: 2022-10-30T14:39:35Z
---

# Spring Boot 源码

## SpringBoot初始化流程

```java
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
    	//资源初始化资源加载器为 null
		this.resourceLoader = resourceLoader;
    	//断言主要加载资源类不能为 null，否则报错
		Assert.notNull(primarySources, "PrimarySources must not be null");
    	//初始化主要加载资源类集合并去重
		this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
    	//推断当前 WEB 应用类型
		this.webApplicationType = WebApplicationType.deduceFromClasspath();
    	//设置应用上下文初始化器
		setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
		//设置监听器	
    	setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
    	//推断主入口应用类
		this.mainApplicationClass = deduceMainApplicationClass();
	}
```
