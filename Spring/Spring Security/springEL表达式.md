---
title: springEL表达式
date: 2022-10-30T15:00:21Z
lastmod: 2022-10-30T15:00:21Z
---

# springEL表达式

　　Spring Security允许我们在定义URL访问或方法访问所应有的权限时使用Spring EL表达式，在定义所需的访问权限时如果对应的表达式返回结果为true则表示拥有对应的权限，反之则无。Spring Security可用表达式对象的基类是SecurityExpressionRoot，其为我们提供了如下在使用Spring EL表达式对URL或方法进行权限控制时通用的内置表达式。

|**表达式**|**描述**|
| ------------------------------| --------------------------------------------------------------------------------------|
|hasRole([role])|当前用户是否拥有指定角色。|
|hasAnyRole([role1,role2])|多个角色是一个以逗号进行分隔的字符串。如果当前用户拥有指定角色中的任意一个则返回true。|
|hasAuthority([auth])|等同于hasRole|
|hasAnyAuthority([auth1,auth2])|等同于hasAnyRole|
|Principle|代表当前用户的principle对象|
|authentication|直接从SecurityContext获取的当前Authentication对象|
|permitAll|总是返回true，表示允许所有的|
|denyAll|总是返回false，表示拒绝所有的|
|isAnonymous()|当前用户是否是一个匿名用户|
|isRememberMe()|表示当前用户是否是通过Remember-Me自动登录的|
|isAuthenticated()|表示当前用户是否已经登录认证成功了。|
|isFullyAuthenticated()|如果当前用户既不是一个匿名用户，同时又不是通过Remember-Me自动登录的，则返回true。|
