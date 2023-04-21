---
title: Spring MVC
date: 2022-10-30T13:59:46Z
lastmod: 2022-10-30T13:59:46Z
---

# Spring MVC

## SpringMVC

### 简单原理图：

　　![](assets/net-img-Nxg2RI-20221030140124-2j3rd9u.png)

### 详细原理图：[

　　](https://imgchr.com/i/Nx2nTe)
![](assets/net-img-1593854466438-89baf576-0e30-4db7-934e-2781413c9510-20221030140124-w8acjeh.png)

### 流程:

1. 浏览器发送请求，到 DispatcherServlet（前端控制器）
2. DispatcherServlet 根据请求信息调用 HandlerMapping（处理器映射器），解析对应 Handler
3. 解析对应 Handler（Controller 控制器）后，开始由 HandlerAdapter 适配器处理。
4. HandlerAdapter 会根据 Handler 来调用处理器处理请求，并处理相应的业务逻辑。
5. 处理完业务后，会返回 ModelAndView 对象
6. ViewResolver 根据逻辑 view 查找到实际 view
7. DispaterServlet 返回 Model 传给 View
8. 把 View 传回浏览器

### 各处理器或控制器作用

1. 前端控制器DispatcherServlet（不需要程序员开发）,由框架提供，在web.xml中配置。
   作用：接收请求，响应结果，相当于转发器，中央处理器。
2. 处理器映射器HandlerMapping(不需要程序员开发),由框架提供。

　　作用：根据请求的url查找Handler(处理器/Controller)，可以通过XML和注解方式来映射。

3. 处理器适配器HandlerAdapter(不需要程序员开发),由框架提供。

　　作用：按照特定规则（HandlerAdapter要求的规则）去执行Handler。

4. 处理器Handler(也称之为Controller，需要工程师开发)

　　注意：编写Handler时按照HandlerAdapter的要求去做，这样适配器才可以去正确执行Handler。
作用：接受用户请求信息，调用业务方法处理请求，也称之为后端控制器。

5. 视图解析器ViewResolver(不需要程序员开发),由框架提供

　　作用：进行视图解析，把逻辑视图名解析成真正的物理视图。
SpringMVC框架支持多种View视图技术，包括：jstlView、freemarkerView、pdfView等。

6. 视图View(需要工程师开发)

　　作用：把数据展现给用户的页面
View是一个接口，实现类支持不同的View技术（jsp、freemarker、pdf等）
