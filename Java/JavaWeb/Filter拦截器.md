---
title: Filter拦截器
date: 2022-10-30T13:27:28Z
lastmod: 2022-10-30T13:27:28Z
---

# Filter拦截器

### 如何实现拦截

　　实现Filter接口，重写doFilter方法。

#### doFilter方法有三个传入参数

- ServletRequest：对于简单的过滤器，大多数过滤逻辑是基于这个对象的。如果处理HTTP请求，并且需要访问诸如getHeader或getCookies等在ServletRequest中无法得到的方法，就要把此对象构造成 HttpServletRequest。
- ServletResponse：除了在两个情形下要使用它以外，通常忽略这个参数。
  - 首先，如果希望完全阻塞对相关 servlet或JSP页面的访问。可调用response.getWriter并直接发送一个响应到客户机。
  - 其次，如果希望修改相关的servlet或 JSP页面的输出，可把响应包含在一个收集所有发送到它的输出的对象中。然后，在调用serlvet或JSP页面后，过滤器可检查输出，如果合适就修改 它，之后发送到客户机。
- FilterChain：对此对象调用doFilter以激活与servlet或JSP页面相关的下一个过滤器。如果没有另一个相关的过滤器，则对doFilter的调用激活servlet或JSP本身
