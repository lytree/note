---
title: Spring Boot 注解
date: 2022-10-30T14:38:32Z
lastmod: 2022-10-30T14:38:32Z
---

# Spring Boot 注解

1. @Configuration(proxyBeanMethods = false) 详解

   1. 如果为true, 则表示被@Bean标识的方法都会被CGLIB进行代理，而且会走bean的生命周期中的一些行为（比如：@PostConstruct,@Destroy等 spring中提供的生命周期）, 如果bean是单例的，那么在同一configuration中调用@Bean标识的方法，无论调用几次得到的都是同一个bean，就是说这个bean只初始化一次。 在该模式下 SpringBoot 每次启动都会判断检查容器中是否存在该组件。
   2. 如果为false,则标识被@Bean标识的方法，不会被拦截进行CGLIB代理，也就不会走bean的生命周期中的一些行为（比如：@PostConstruct,@Destroy等 spring中提供的生命周期），如果同一个configuration中调用@Bean标识的方法，就只是普通方法的执行而已，并不会从容器中获取对象。所以如果单独调用@Bean标识的方法就是普通的方法调用，而且不走bean的生命周期。在该模式下 SpringBoot 每次启动会跳过检查容器中是否存在该组件。
2. @ConditionalOnBean：当容器里有指定Bean的条件下
3. @ConditionalOnClass：当类路径下有指定的类的条件下
4. @ConditionalOnExpression：基于SpEL表达式为true的时候作为判断条件才去实例化
5. @ConditionalOnJava：基于JVM版本作为判断条件
6. @ConditionalOnJndi：在JNDI存在的条件下查找指定的位置
7. @ConditionalOnMissingBean：当容器里没有指定Bean的情况下
8. @ConditionalOnMissingClass：当容器里没有指定类的情况下
9. @ConditionalOnWebApplication：当前项目时Web项目的条件下
10. @ConditionalOnNotWebApplication：当前项目不是Web项目的条件下
11. @ConditionalOnProperty：指定的属性是否有指定的值
12. @ConditionalOnResource：类路径是否有指定的值
13. @ConditionalOnOnSingleCandidate：当指定Bean在容器中只有一个，或者有多个但是指定首选的Bean
