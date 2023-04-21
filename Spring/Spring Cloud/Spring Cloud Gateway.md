---
title: Spring Cloud Gateway
date: 2022-10-30T14:41:11Z
lastmod: 2022-10-30T14:41:11Z
---

# Spring Cloud Gateway

## Spring Cloud Gateway的处理流程

　　客户端向 Spring Cloud Gateway 发出请求。然后在 Gateway Handler Mapping 中找到与请求相匹配的路由，将其发送到 Gateway Web Handler。Handler 再通过指定的过滤器链来将请求发送到我们实际的服务执行业务逻辑，然后返回。过滤器之间用虚线分开是因为过滤器可能会在发送代理请求之前（“pre”）或之后（“post”）执行业务逻辑。
![19816137-eeedbd49be096c05.jpg](https://cdn.nlark.com/yuque/0/2021/jpeg/381674/1613306279918-0545edd7-5ec4-4545-9535-406dc012aa5f.jpeg#align=left&display=inline&height=595&margin=%5Bobject%20Object%5D&name=19816137-eeedbd49be096c05.jpg&originHeight=595&originWidth=443&size=19713&status=done&style=none&width=443)

## 路由配置方式

### 和注册中心相结合的路由配置方式

　　基础URI路由配置方式
如果请求的目标地址，是单个的URI资源路径，配置文件示例如下：

```yaml
spring:
  application:
    name: api-gateway
  cloud:
    gateway:
      routes:
        -id: url-proxy-1
          uri: https://www.baidu.com
          predicates:
            -Path=/baidu
```

　　id：我们自定义的路由 ID，保持唯一
uri：目标服务地址
predicates：路由条件，Predicate 接受一个输入参数，返回一个布尔值结果。该接口包含多种默认方法来将 Predicate 组合成其他复杂的逻辑（比如：与，或，非）。

### 和注册中心相结合的路由配置方式

　　在uri的schema协议部分为自定义的lb:类型，表示从微服务注册中心订阅服务，并且进行服务的路由。

```yaml
spring:
  cloud:
    gateway:
      routes:
      -id: seckill-provider-route
        uri: lb://seckill-provider
        predicates:
        - Path=/seckill-provider/**

      -id: message-provider-route
        uri: lb://message-provider
        predicates:
        -Path=/message-provider/**
```

　　注册中心相结合的路由配置方式，与单个URI的路由配置，区别其实很小，仅仅在于URI的schema协议不同。单个URI的地址的schema协议，一般为http或者https协议。

## 路由 匹配规则

　　Spring Cloud Gateway 是通过 Spring WebFlux 的 HandlerMapping 做为底层支持来匹配到转发路由，Spring Cloud Gateway 内置了很多 Predicates 工厂，这些 Predicates 工厂通过不同的 HTTP 请求参数来匹配，多个 Predicates 工厂可以组合使用。
![20200527213652534.png](https://cdn.nlark.com/yuque/0/2021/png/381674/1613306618178-bf8c748c-70e2-4601-8803-ef67ee742f3f.png#align=left&display=inline&height=588&margin=%5Bobject%20Object%5D&name=20200527213652534.png&originHeight=588&originWidth=1249&size=91303&status=done&style=none&width=1249)
gateWay的主要功能之一是转发请求，转发规则的定义主要包含三个部分

|Route（路由）|路由是网关的基本单元，由ID、URI、一组Predicate、一组Filter组成，根据Predicate进行匹配转发。|
| -----------------------| ---------------------------------------------------------------------------------------------------------------------------|
|Predicate（谓语、断言）|路由转发的判断条件，目前SpringCloud Gateway支持多种方式，常见如：Path、Query、Method、Header等，写法必须遵循 key=vlue的形式|
|Filter（过滤器）|过滤器是路由转发请求时所经过的过滤逻辑，可用于修改请求、响应内容|

> 其中Route和Predicate必须同时申明

### Predicate 断言条件(转发规则)介绍

　　Predicate 来源于 Java 8，是 Java 8 中引入的一个函数，Predicate 接受一个输入参数，返回一个布尔值结果。该接口包含多种默认方法来将 Predicate 组合成其他复杂的逻辑（比如：与，或，非）。可以用于接口请求参数校验、判断新老数据是否有变化需要进行更新操作。
在 Spring Cloud Gateway 中 Spring 利用 Predicate 的特性实现了各种路由匹配规则，有通过 Header、请求参数等不同的条件来进行作为条件匹配到对应的路由。网上有一张图总结了 Spring Cloud 内置的几种 Predicate 的实现
![19816137-bb046dbf19bee1b4.gif](https://cdn.nlark.com/yuque/0/2021/gif/381674/1613306841584-1862d5d1-9cea-4e9c-8cb1-f4b0b55819cf.gif#align=left&display=inline&height=361&margin=%5Bobject%20Object%5D&name=19816137-bb046dbf19bee1b4.gif&originHeight=361&originWidth=692&size=71052&status=done&style=none&width=692)
Spring Cloud GateWay 内置几种 Predicate 的使用。

|规则|实例|说明|
| :------| :----------------------------------------------------------------------------------------------------| :-----------------------------------------------------------------------|
|Path|- Path=/gate/**,/rule/**|## 当请求的路径为gate、rule开头的时，转发到http://localhost:9023服务器上|
|Before|- Before=2017-01-20T17:42:47.789-07:00[America/Denver]|在某个时间之前的请求才会被转发到 http://localhost:9023服务器上|
|After|- After=2017-01-20T17:42:47.789-07:00[America/Denver]|在某个时间之后的请求才会被转发|
|Between|- Between=2017-01-20T17:42:47.789-07:00[America/Denver],2017-01-21T17:42:47.789-07:00[America/Denver]|在某个时间段之间的才会被转发|
|Cookie|- Cookie=chocolate, ch.p|名为chocolate的表单或者满足正则ch.p的表单才会被匹配到进行请求转发|
|Header|- Header=X-Request-Id, \d+|携带参数X-Request-Id或者满足\d+的请求头才会匹配|
|Host|- Host=www.hd123.com|当主机名为www.hd123.com的时候直接转发到http://localhost:9023服务器上|
|Method|- Method=GET|只有GET方法才会匹配转发请求，还可以限定POST、PUT等请求方式|
|Query| -Query=keep, pu.|localhost:8080?keep=pub 只有keep等于参数符合是才匹配路由|

```yaml
spring:
  application:
    name: api-gateway
  cloud:
    gateway:
      routes:
        - id: gateway-service
          uri: https://www.baidu.com
          order: 0
          predicates:
            - Host=**.foo.org
            - Path=/headers
            - Method=GET
            - Header=X-Request-Id, \d+
            - Query=foo, ba.
            - Query=baz
            - Cookie=chocolate, ch.p
```

### 过滤器规则（Filter）

|过滤规则|实例|说明|
| :------------------| :-----------------------------| :--------------------------------------------------------------------|
|PrefixPath|- PrefixPath=/app|在请求路径前加上app|
|RewritePath|- RewritePath=/test, /app/test|访问localhost:9022/test,请求会转发到localhost:8001/app/test|
|SetPath|SetPath=/app/{path}|通过模板设置路径，转发的规则时会在路径前增加app，{path}表示原请求路径|
|RedirectTo||重定向|
|RemoveRequestHeader||去掉某个请求头信息|

　　注：当配置多个filter时，优先定义的会被调用，剩余的filter将不会生效

### 实现熔断降级

## 高级配置

### 分布式限流

　　令牌桶算法

　　不同限流方案
根据用户ID限流，请求路径中必须携带userId参数

```java
@Bean
KeyResolver userKeyResolver() {
  return exchange -> Mono.just(exchange.getRequest().getQueryParams().getFirst("user"));
}
```

　　KeyResolver需要实现resolve方法，比如根据userid进行限流，则需要用userid去判断。实现完KeyResolver之后，需要将这个类的Bean注册到Ioc容器中。
如果需要根据IP限流，定义的获取限流Key的bean为：

```java
@Bean
public KeyResolver ipKeyResolver() {
  return exchange -> Mono.just(exchange.getRequest().getRemoteAddress().getHostName());
}
```

　　通过exchange对象可以获取到请求信息，这边用了HostName，如果你想根据用户来做限流的话这边可以获取当前请求的用户ID或者用户名就可以了，比如：
如果需要根据接口的URI进行限流，则需要获取请求地址的uri作为限流key，定义的Bean对象为：

```java
@Bean
KeyResolver apiKeyResolver() {
  return exchange -> Mono.just(exchange.getRequest().getPath().value());
}
```

　　 通过exchange对象可以获取到请求信息，这边用了HostName，如果你想根据用户来做限流的话这边可以获取当前请求的用户ID或者用户名就可以了，比如：如果需要根据接口的URI进行限流，则需要获取请求地址的uri作为限流key，定义的Bean对象为：

```java
@Bean
KeyResolver apiKeyResolver() {
  return exchange -> Mono.just(exchange.getRequest().getPath().value());
}
```

### 统一配置跨域请求

```java
spring:
  cloud:
    gateway:
      globalcors:
        cors-configurations:
          '[/**]':
            allowed-origins: "*"
            allowed-headers: "*"
            allow-credentials: true
            allowed-methods:
              - GET
              - POST
              - DELETE
              - PUT
              - OPTION
```
