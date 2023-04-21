---
title: spring 注解
date: 2022-10-30T14:00:20Z
lastmod: 2022-10-30T14:00:20Z
---

# spring 注解

## [@RestController ](/RestController ) vs [@Controller ](/Controller )

　　`Controller` 返回一个页面,单独使用 [@Controller ](/Controller ) 不加 [@ResponseBody ](/ResponseBody ) 的话一般使用在要返回一个视图
`@RestController`= `@Controller` +`@ResponseBody` 返回 JSON 或 XML 形式数据,但`@RestController`只返回对象，对象数据直接以 JSON 或 XML 形式写入 HTTP 响应(Response)中，
`@ResponseBody` 注解的作用是将 `Controller` 的方法返回的对象通过适当的转换器转换为指定的格式之后，写入到 HTTP 响应(Response)对象的 body 中，通常用来返回 JSON 或者 XML 数据，返回 JSON 数据的情况比较多。

## [@Component ](/Component ) 和[@Bean ](/Bean ) 的区别

1. [@Component ](/Component ) 注解作用于类，[@Bean ](/Bean ) 注解作用于方法
2. [@Component ](/Component ) 通常用于类路径扫描自动侦测以及装配到 Spring 容器中我们可以使用 [@ComponentScan ](/ComponentScan ) 注解定义要扫描的路径从中找出标识了需要装配的类自动装配到 Spring 的 bean 容器中），[@Bean ](/Bean ) 注解通常是在标有该注解的方法中定义一个 Bean
3. [@Bean ](/Bean ) 注解比[@Component ](/Component ) 注解自定义性更强，并且在引用第三方库中的类需要装配到 spring 容器时，只能通过[@Bean ](/Bean ) 实现

## 将一个类声明为 Spring 的 Bean 的注解

　　一般使用 `@Autowired ​` 注解自动装配 bean，要想把类标识成可用于```@Autowired` 注解自动装配的 bean 的类,采用以下注解可实现：

- `@Component ​` ：通用的注解，可标注任意类为 Spring 组件。如果一个 Bean 不知道属于哪个层，可以使用`@Component`  注解标注。
- `@Repository`  : 对应持久层即 Dao 层，主要用于数据库相关操作。
- `@Service ​` : 对应服务层，主要涉及一些复杂的逻辑，需要用到 Dao 层。
- `@Controller`  : 对应 Spring MVC 控制层，主要用户接受用户请求并调用 Service 层返回数据给前端页面。

#### 注意事项

　　`@Autowired` 自动装配时，先根据类型去获取`bean`，若类型的bean有多个会根据@Autowire装配的名字去获取，若根据名字未找到，则报错。

### @Import()

　　将普通类注入IOC容器中
