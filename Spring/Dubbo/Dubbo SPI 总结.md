---
title: Dubbo SPI 总结
date: 2022-12-30T20:46:22Z
lastmod: 2022-12-30T20:46:22Z
---

# Dubbo SPI 总结

---

### ExtensionLoader

### 主要属性

#### EXTENSION_LOADERS

　　​`ConcurrentMap`​类型，存放扩展点接口与扩展点加载器的对应关系：

　　​`Protocol.class`​ ->  `Protocol的ExtensionLoader实例对象`​

　　​`ExtensionFactory.class`​  ->  `ExtensionFactory的ExtensionLoader实例对象`​

#### EXTENSION_INSTANCES

　　​`ConcurrentMap`​类型，存放扩展点类与扩展点类实例的对应关系：

　　​`DubboProtocol.class`​ ->  `DubboProtocol实例对象`​

　　​`SpiExtensionFactory.class`​  ->  `SpiExtensionFactory实例对象`​

#### type

　　当前ExtensionLoader实例是哪个接口的扩展点加载器，如：`Protocol.class`​

#### objectFactory

　　对象工厂，用来获取对象，获取的对象可能是DubboSPI机制产生的，也可能是从Spring容器或其他容器中获取的。

#### cachedInstances

　　扩展点名字与扩展点实例对应关系

　　​`dubbo`​ ->  `Holder<DubboProtocol>`​

#### cachedDefaultName

　　默认扩展点的名字，如`@SPI("dubbo")`​的dubbo

#### cachedWrapperClasses

　　缓存扩展点的包装类

### 主要方法

#### getExtension

　　getExtension(String name)：用来获取`name`​对应的扩展点实例对象，获取一次后，会被缓存，下次获取直接从缓存里拿。

#### getAdaptiveExtension

　　getAdaptiveExtension()：获取一个自适应的扩展点实例

#### createExtension

　　在创建一个扩展点实例时，过程如下：

1. 查找缓存`cachedClasses`​是否已经有已经缓存的的扩展点class。
2. 如果没有则解析接口文件（`META-INF/dubbo/internal/`​、`META-INF/dubbo/`​、`META-INF/services/`​），找到对应的扩展点class。
3. 从解析的class中，找到对应的扩展点实现类
4. 找到这个实现类后，生成一个实例对象，并放入缓存中。
5. 根据实现类生成一个实例，把实现类和对应生成的实例放入缓存中，方便下次获取
6. 对生成的实例对象进行属性赋值操作
7. 判断是否存在Wrapper包装类
8. 如果存在Wrapper包装类，则进行实例化并进行属性赋值
9. 如果有多个Wrapper，则会对多个Wrapper进行一层一层包装并赋值
10. 如果是Lifecycle类型，则进行初始化操作
11. 返回最终的Wrapper实例对象

#### loadExtensionClasses

　　​`loadExtensionClasses()`​方法加载当前接口类型对应的所有扩展点实现类的，返回值类型是Map类型。返回的这个map最终放入`cachedClasses`​缓存起来，下次获取直接从缓存中获取。

　　Dubbo在加载扩展点时，有3种读取策略，分别为`DubboInternalLoadingStrategy`​、`DubboLoadingStrategy`​、`ServicesLoadingStrategy`​。这3个类分别负责读取的文件目录为`META-INF/dubbo/internal/`​、`META-INF/dubbo/`​、`META-INF/services/`​。

　　读取时到3个目录寻找接口全限定名文件，再调用loadResource方法进行加载。

#### loadResource

　　该方法是对文件内容的解析，一行一行的解析，把解析出的内容按"="进行分隔，"="前面是扩展点的名字，后面是对应的实现类，拿到实现类后，用ExtensionLoader类的类加载器来加载扩展点实现类。

　　再调用`loadClass`​方法对扩展点实例进行详细的解析，把解析后的最终对象放到map中缓存起来。

#### loadClass

　　​`loadClass`​方法处理过程如下：

1. 判断当前扩展点实现类，是否实现了扩展点接口，如果没有实现则抛出异常
2. 判断当前扩展点实现类上是否被`@Adaptive`​注解标记。
3. 如果存在的话，说明该类是当前接口的默认自适应类，并把该类赋值给`cachedAdaptiveClass`​，作为缓存。
4. 判断当前扩展点实现类是否为`Wrapper`​类，如何是的话，则把该类添加到`cachedWrapperClasses`​这个Set集合中，`Wrapper`​类可能存在多个，有多个添加多个。
5. 判断当前扩展点实现类是否存在无参构造方法，没有则报错。
6. 判断是否存在name，如果没有name，查看是否存在`Extension`​注解。
7. 存在`Extension`​注解，获取注解的值作为name。
8. 如果没有`Extension`​注解获取类名处理后作为name。
9. 如果没有name，则报错。
10. 如果有多个name，则判断当前扩展点实现类上是否存在`@Activate`​注解。
11. 如果存在`@Activate`​注解，则把该类添加到`cachedActivates`​缓存中。
12. 如果有多个name，则把每个name和对应的实现类存到`extensionClasses`​中，最终会被放到`cachedClasses`​中缓存起来。

#### injectExtension

　　此方法为Dubbo的IOC，处理过程如下：

1. 查找当前实例类中的setter方法
2. 判断setter是否通过注解`DisableInject`​禁用了依赖注入
3. 判断setter方法的参数类型，如果是基本类型则跳过
4. 通过set方法名，获取要对应的属性值
5. 调用objectFactory实例的`getExtension`​方法，根据参数类型和属性来获取对象，这个对象可能来着spring容器或者DubboSPI机制得到的对象。
6. 调用方法对象的invoke方法来进行赋值操作。

#### createAdaptiveExtensionClass

　　Dubbo的自适应扩展点对象是通过Dubbo内部生成代理类，通过编译产生的代理对象。

　　Dubbo可以通过`@Adaptive`​注解类为某个接口生成代理类。Dubbo通过内部的代码模板生成一个类，然后编译产生一个代理对象。

　　使用`@Adaptive`​注解产生代理类是有一定条件的。被`@Adaptive`​注解的方法，必须有URL参数或者参数类有URL属性。

## 疑问

1. Wrapper类可以指定顺序吗？

　　Wrapper类是不支持指定顺序的，不是说谁写在前面谁就在里面，Dubbo内部会对Wrapper类进行排序的，在`org.apache.dubbo.common.extension.ExtensionLoader#createExtension`​​方法中有体现。

　　如果有`@Activate`​​注解，可以通过注解的`order`​​属性执行顺序。

```java
List<Class<?>> wrapperClassesList = new ArrayList<>();
if (cachedWrapperClasses != null) {
    wrapperClassesList.addAll(cachedWrapperClasses);
    wrapperClassesList.sort(WrapperComparator.COMPARATOR);
    Collections.reverse(wrapperClassesList);
}
```

2. Dubbo的依赖注入会出现循环依赖吗？

　　Dubbo的依赖注入是不会出现循环依赖的，因为Dubbo在注入扩展点属性的时候，赋值的是一个`Adaptive`​​代理类，而不是我们写的普通类。

　　‍
