---
title: SPI机制
date: 2022-12-16T12:56:14Z
lastmod: 2022-12-30T21:10:50Z
---

# SPI机制

　　​`SPI`​是英文`Service Provider Interface`​ 的缩写，是一种服务发现机制。`SPI`​是一种可插拔机制，需要定义一个接口，然后针对不同的场景进行实现。

# JAVA SPI

　　​`JAVA`​自带的`SPI`​机制是`JDK1.6`​引入的。`Java`​的`SPI`​约定服务的提供方需要在资源路径下的`META-INF/services`​文件夹下面以服务的接口为文件名提供服务接口实现类，自动加载文件里所定义的类

　　例如 数据库驱动加载

```java
//DriverManager


public class DriverManager {
...
    private static void loadInitialDrivers() {
        String drivers;
        try {
	    // 加载配置文件中的jdbc.drivers指定的驱动
            drivers = AccessController.doPrivileged(new PrivilegedAction<String>() {
                public String run() {
                    return System.getProperty("jdbc.drivers");
                }
            });
        } catch (Exception ex) {
            drivers = null;
        }
        // If the driver is packaged as a Service Provider, load it.
        // Get all the drivers through the classloader
        // exposed as a java.sql.Driver.class service.
        // ServiceLoader.load() replaces the sun.misc.Providers()
        // 加载SPI约定的/META-INF/services下面的驱动
        AccessController.doPrivileged(new PrivilegedAction<Void>() {
            public Void run() {
		// java.sql.Driver
                ServiceLoader<Driver> loadedDrivers = ServiceLoader.load(Driver.class);
                Iterator<Driver> driversIterator = loadedDrivers.iterator();

                /* Load these drivers, so that they can be instantiated.
                 * It may be the case that the driver class may not be there
                 * i.e. there may be a packaged driver with the service class
                 * as implementation of java.sql.Driver but the actual class
                 * may be missing. In that case a java.util.ServiceConfigurationError
                 * will be thrown at runtime by the VM trying to locate
                 * and load the service.
                 *
                 * Adding a try catch block to catch those runtime errors
                 * if driver not available in classpath but it's
                 * packaged as service and that service is there in classpath.
                 */
                try{
		    // 迭代器中循环加载驱动
                    while(driversIterator.hasNext()) {
                        driversIterator.next();
                    }
                } catch(Throwable t) {
                // Do nothing
                }
                return null;
            }
        });

        println("DriverManager.initialize: jdbc.drivers = " + drivers);

        if (drivers == null || drivers.equals("")) {
            return;
        }
        String[] driversList = drivers.split(":");
        println("number of Drivers:" + driversList.length);
        for (String aDriver : driversList) {
            try {
                println("DriverManager.Initialize: loading " + aDriver);
                Class.forName(aDriver, true,
                        ClassLoader.getSystemClassLoader());
            } catch (Exception ex) {
                println("DriverManager.Initialize: load failed: " + ex);
            }
        }
    }
}
```

```java
//ServiceLoader
public final class ServiceLoader<S>
    implements Iterable<S>{
	    // java.util.ServiceLoader.LazyIterator#hasNextService
	    private boolean hasNextService() {
	            if (nextName != null) {
	                return true;
	            }
	            if (configs == null) {
	                try {
	                	// 查找META-INF/services/下面的指定service（java.sql.Driver）的文件
	                	// fullName = META-INF/services/java.sql.Driver
	                    String fullName = PREFIX + service.getName();
	                    if (loader == null)
	                        configs = ClassLoader.getSystemResources(fullName);
	                    else
	                        configs = loader.getResources(fullName);
	                } catch (IOException x) {
	                    fail(service, "Error locating configuration files", x);
	                }
	            }
	            while ((pending == null) || !pending.hasNext()) {
	                if (!configs.hasMoreElements()) {
	                    return false;
	                }
	                pending = parse(service, configs.nextElement());
	            }
	            nextName = pending.next();
	            return true;
	        }

	    private S nextService() {
            	if (!hasNextService())
	                throw new NoSuchElementException();
	            String cn = nextName;
	            nextName = null;
	            Class<?> c = null;
	            try {
	           	// 加载数据库驱动
	                c = Class.forName(cn, false, loader);
	            } catch (ClassNotFoundException x) {
	                fail(service,
	                     "Provider " + cn + " not found");
	            }
	            if (!service.isAssignableFrom(c)) {
	                fail(service,
	                     "Provider " + cn  + " not a subtype");
	            }
	            try {
	                S p = service.cast(c.newInstance());
	                providers.put(cn, p);
	                return p;
	            } catch (Throwable x) {
	                fail(service,
	                     "Provider " + cn + " could not be instantiated",
	                     x);
	            }
	            throw new Error(); 
	        }
}
```

# Dubbo SPI

　　​`JAVA SPI`​不能按需加载，接口对应的文件里有多少个类，就会实例化多少，所以Dubbo自己实现类一套SPI机制，即`Dubbo SPI`​，`Dubbo SPI`​ 支持 `依赖注入`​ 操作

## 2.7

### 源码解析

#### 获取 `@SPI`​修饰 接口对应的扩展点加载器。

```java
//ExtensionLoader.getExtensionLoader 
    public static <T> ExtensionLoader<T> getExtensionLoader(Class<T> type) {
        if (type == null) {
            throw new IllegalArgumentException("Extension type == null");
        }
        if (!type.isInterface()) {
            throw new IllegalArgumentException("Extension type (" + type + ") is not an interface!");
        }
        if (!withExtensionAnnotation(type)) {
            throw new IllegalArgumentException("Extension type (" + type +
                    ") is not an extension, because it is NOT annotated with @" + SPI.class.getSimpleName() + "!");
        }

        ExtensionLoader<T> loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
        if (loader == null) {
            EXTENSION_LOADERS.putIfAbsent(type, new ExtensionLoader<T>(type));
            loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
        }
        return loader;
    }
```

#### 获取一个具体的扩展点

```java
    public T getExtension(String name) {
        return getExtension(name, true);
    }

    public T getExtension(String name, boolean wrap) {
        if (StringUtils.isEmpty(name)) {
            throw new IllegalArgumentException("Extension name == null");
        }
        if ("true".equals(name)) {
            return getDefaultExtension();
        }
        final Holder<Object> holder = getOrCreateHolder(name);//getOrCreateHolder 获取class对应的holder对象，这个对象作为一个锁，跟并发有关系，防止重复生成。
        Object instance = holder.get();
        if (instance == null) {
            synchronized (holder) {
                instance = holder.get();
                if (instance == null) {
                    instance = createExtension(name, wrap);//createExtension方法是创建扩展点。
                    holder.set(instance);
                }
            }
        }
        return (T) instance;
    }
```

#### 创建一个扩展点

```java
    private T createExtension(String name, boolean wrap) {
	/**
	*  getExtensionClasses方法是获取解析配置文件，查找所有的扩展点并缓存起来。 
	*  查找配置文件的路径为META-INF/dubbo/internal/、META-INF/dubbo/、META-INF/services/。
        *  查找到之后，把对应的key和类class文件放到map中。再根据传进来的name获取对应的class文件，并进行实例化。
	**/
        Class<?> clazz = getExtensionClasses().get(name);
        if (clazz == null || unacceptableExceptions.contains(name)) {
            throw findException(name);
        }
        try {
            T instance = (T) EXTENSION_INSTANCES.get(clazz);
            if (instance == null) {
                EXTENSION_INSTANCES.putIfAbsent(clazz, clazz.getDeclaredConstructor().newInstance());
                instance = (T) EXTENSION_INSTANCES.get(clazz);
            }
            injectExtension(instance); //进行实例化


            if (wrap) {//对实例化后的对象，看里面有哪些属性并进行赋值。

                List<Class<?>> wrapperClassesList = new ArrayList<>();
                if (cachedWrapperClasses != null) {
                    wrapperClassesList.addAll(cachedWrapperClasses);
                    wrapperClassesList.sort(WrapperComparator.COMPARATOR);
                    Collections.reverse(wrapperClassesList);
                }

                if (CollectionUtils.isNotEmpty(wrapperClassesList)) {
                    for (Class<?> wrapperClass : wrapperClassesList) {
                        Wrapper wrapper = wrapperClass.getAnnotation(Wrapper.class);
                        if (wrapper == null
                                || (ArrayUtils.contains(wrapper.matches(), name) && !ArrayUtils.contains(wrapper.mismatches(), name))) {
                            instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
                        }
                    }
                }
            }

            initExtension(instance);
            return instance;
        } catch (Throwable t) {
            throw new IllegalStateException("Extension instance (name: " + name + ", class: " +
                    type + ") couldn't be instantiated: " + t.getMessage(), t);
        }
    }

```

#### 获取扩展点类 loadExtensionClasses

```java
    /**
     * synchronized in getExtensionClasses
     */
    private Map<String, Class<?>> loadExtensionClasses() {
        cacheDefaultExtensionName();//缓存默认扩展点

        Map<String, Class<?>> extensionClasses = new HashMap<>();

        for (LoadingStrategy strategy : strategies) {//根据某种策略来读取对应的文件夹。读取文件里的文件找到类，访问map中，并且返回。
            loadDirectory(extensionClasses, strategy.directory(), type.getName(), strategy.preferExtensionClassLoader(),
                    strategy.overridden(), strategy.excludedPackages());
            loadDirectory(extensionClasses, strategy.directory(), type.getName().replace("org.apache", "com.alibaba"),
                    strategy.preferExtensionClassLoader(), strategy.overridden(), strategy.excludedPackages());
        }

        return extensionClasses;
    }
    /**
     * extract and cache default extension name if exists
     * 缓存默认扩展点
     */
    private void cacheDefaultExtensionName() {
        // @SPI("xxx")
        final SPI defaultAnnotation = type.getAnnotation(SPI.class);
        if (defaultAnnotation == null) {
            return;
        }

        String value = defaultAnnotation.value();
        if ((value = value.trim()).length() > 0) {
            String[] names = NAME_SEPARATOR.split(value);
            if (names.length > 1) {
                throw new IllegalStateException("More than 1 default extension name on extension " + type.getName()
                        + ": " + Arrays.toString(names));
            }
            if (names.length == 1) {
                cachedDefaultName = names[0];
            }
        }
    }

    // 读取策略
    /**
    * org.apache.dubbo.common.extension.LoadingStrategy
    * org.apache.dubbo.common.extension.DubboInternalLoadingStrategy
    * org.apache.dubbo.common.extension.DubboLoadingStrategy
    * org.apache.dubbo.common.extension.ServicesLoadingStrategy
    *  也就是这3种策略来读取文件。打开3个类的内容，这3个类分别对应3个文件夹，内容比较简单。
    *   DubboInternalLoadingStrategy类对应META-INF/dubbo/internal/。
    *   DubboLoadingStrategy类对应META-INF/dubbo/。
    *   ServicesLoadingStrategy类对应META-INF/services/。
    *
    *
    *
    **/
    private static volatile LoadingStrategy[] strategies = loadLoadingStrategies();
    private static LoadingStrategy[] loadLoadingStrategies() {
    return StreamSupport.stream(ServiceLoader.load(LoadingStrategy.class)
        .spliterator(), false)
        .sorted()
        .toArray(LoadingStrategy[]::new);
    }
    // 读取目录文件内容
    private void loadDirectory(Map<String, Class<?>> extensionClasses, String dir, String type,
                               boolean extensionLoaderClassLoaderFirst, boolean overridden, String... excludedPackages) {
        String fileName = dir + type;
        try {
            Enumeration<java.net.URL> urls = null;
            ClassLoader classLoader = findClassLoader();
            //findClassLoader(); 首先查看当前线程的类加载器，如果没有的话，就找ExtensionLoader这个类是由哪个类加载器进行加载的，如果还没有的话，就返回系统类加载器。

            // try to load from ExtensionLoader's ClassLoader first
            if (extensionLoaderClassLoaderFirst) {
                ClassLoader extensionLoaderClassLoader = ExtensionLoader.class.getClassLoader();
                if (ClassLoader.getSystemClassLoader() != extensionLoaderClassLoader) {
                    urls = extensionLoaderClassLoader.getResources(fileName);
                }
            }

            if (urls == null || !urls.hasMoreElements()) {
                if (classLoader != null) {
                    urls = classLoader.getResources(fileName);
                } else {
                    urls = ClassLoader.getSystemResources(fileName);
                }
            }

            if (urls != null) {
                while (urls.hasMoreElements()) {
                    java.net.URL resourceURL = urls.nextElement();
                    loadResource(extensionClasses, classLoader, resourceURL, overridden, excludedPackages);
                }
            }
        } catch (Throwable t) {
            logger.error("Exception occurred when loading extension class (interface: " +
                    type + ", description file: " + fileName + ").", t);
        }
    }
    //读取文件内容
    private void loadResource(Map<String, Class<?>> extensionClasses, ClassLoader classLoader,
                              java.net.URL resourceURL, boolean overridden, String... excludedPackages) {
        try {
            try (BufferedReader reader = new BufferedReader(new InputStreamReader(resourceURL.openStream(), StandardCharsets.UTF_8))) {
                String line;
                String clazz = null;
                while ((line = reader.readLine()) != null) {
                    final int ci = line.indexOf('#');
                    if (ci >= 0) {
                        line = line.substring(0, ci);
                    }
                    line = line.trim();
                    if (line.length() > 0) {
                        try {
                            String name = null;
                            int i = line.indexOf('=');
                            if (i > 0) {
                                name = line.substring(0, i).trim();
                                clazz = line.substring(i + 1).trim();
                            } else {
                                clazz = line;
                            }
                            if (StringUtils.isNotEmpty(clazz) && !isExcluded(clazz, excludedPackages)) {
                                loadClass(extensionClasses, resourceURL, Class.forName(clazz, true, classLoader), name, overridden);
                            }
                        } catch (Throwable t) {
                            IllegalStateException e = new IllegalStateException(
                                    "Failed to load extension class (interface: " + type + ", class line: " + line + ") in " + resourceURL +
                                            ", cause: " + t.getMessage(), t);
                            exceptions.put(line, e);
                        }
                    }
                }
            }
        } catch (Throwable t) {
            logger.error("Exception occurred when loading extension class (interface: " +
                    type + ", class file: " + resourceURL + ") in " + resourceURL, t);
        }
    }
    //首先判断传进来的类，是否实现了type这个接口，如果没有实现则抛出异常。再看这个类是否存在Adaptive注解，如果存在，则缓存一下。 
    //判断这个类是否是一个Wrapper类，判断逻辑就是查看这个类是否存在一个含有type接口的构造函数。
    // 我们看到一个获取构造方法的代码clazz.getConstructor()并没有变量接收这是为啥？
    //这是因为这些扩展点类，需要一个无参的构造函数，如果没有的话，会直接报错。
    //如果这个类没有对应的name，则查看这个类有没有Extension注解，有的话取这个注解的值作为name，如果没有则用小写的类名。
  
    private void loadClass(Map<String, Class<?>> extensionClasses, java.net.URL resourceURL, Class<?> clazz, String name,
                           boolean overridden) throws NoSuchMethodException {
        if (!type.isAssignableFrom(clazz)) {
            throw new IllegalStateException("Error occurred when loading extension class (interface: " +
                    type + ", class line: " + clazz.getName() + "), class "
                    + clazz.getName() + " is not subtype of interface.");
        }
        if (clazz.isAnnotationPresent(Adaptive.class)) {
            cacheAdaptiveClass(clazz, overridden);
        } else if (isWrapperClass(clazz)) {
            cacheWrapperClass(clazz);
        } else {
            clazz.getConstructor();
            if (StringUtils.isEmpty(name)) {
                name = findAnnotationName(clazz);
                if (name.length() == 0) {
                    throw new IllegalStateException(
                            "No such extension name for the class " + clazz.getName() + " in the config " + resourceURL);
                }
            }

            String[] names = NAME_SEPARATOR.split(name);
            if (ArrayUtils.isNotEmpty(names)) {
                cacheActivateClass(clazz, names[0]);
                for (String n : names) {
                    cacheName(clazz, n);
                    saveInExtensionClass(extensionClasses, clazz, n, overridden);
                }
            }
        }
    }

```

#### 依赖注入

```java
    private T injectExtension(T instance) {

        if (objectFactory == null) {
            return instance;
        }

        try {//首先先遍历类里面的所有方法。
            for (Method method : instance.getClass().getMethods()) {
                if (!isSetter(method)) {//判断是否是setter方法，从这里可以看出是根据set方法进行赋值的。
                    continue;
                }

                /*
                 * Check {@link DisableInject} to see if we need autowire injection for this property
                 *///判断方法上是否有DisableInject注解，可以通过这个注解来忽略赋值。
                if (method.getAnnotation(DisableInject.class) != null) {
                    continue;
                }
                //判断是否是setter方法的参数类型，如果是基础类型也会直接忽略。
                Class<?> pt = method.getParameterTypes()[0];
                if (ReflectUtils.isPrimitives(pt)) {
                    continue;
                }

                /*
                 * Check {@link Inject} to see if we need auto-injection for this property
                 * {@link Inject#enable} == false will skip inject property phase
                 * {@link Inject#InjectType#ByName} default inject by name
                 *///获取属性名称，也就是set方法去掉set再把第一个字符小写。
                String property = getSetterProperty(method);
            
                Inject inject = method.getAnnotation(Inject.class);
                if (inject == null) {
                    injectValue(instance, method, pt, property);
                } else {
                    if (!inject.enable()) {
                        continue;
                    }

                    if (inject.type() == Inject.InjectType.ByType) {
                        injectValue(instance, method, pt, null);
                    } else {
                        injectValue(instance, method, pt, property);
                    }
                }
            }
        } catch (Exception e) {
            logger.error(e.getMessage(), e);
        }
        return instance;
    }
    //从对象工厂objectFactory里获取对应的值，赋值给对应的属性。
    private void injectValue(T instance, Method method, Class<?> pt, String property) {
        try {
            Object object = objectFactory.getExtension(pt, property);
            if (object != null) {
                method.invoke(instance, object);
            }
        } catch (Exception e) {
            logger.error("Failed to inject via method " + method.getName()
                    + " of interface " + type.getName() + ": " + e.getMessage(), e);
        }
    }

```

##### objectFactory

　　以上几步就是Dubbo对扩展点属性复制的过程，Dubbo支持从多种容器里获取对象进行赋值，即`objectFactory`​可以获取Dubbo生成的代理对象也可以从Spring容器中获取对象给属性复制。因此，`objectFactory`​对应的类，也有SPI注解。

```java
@SPI
public interface ExtensionFactory {
    <T> T getExtension(Class<T> type, String name);
}
复制代码
```

　　​`objectFactory`​的初始化是在`ExtensionLoader`​的构造方法中。

```ini
private ExtensionLoader(Class<?> type) {
    this.type = type;
    objectFactory =
            (type == ExtensionFactory.class ? null : 
            ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension());
}
复制代码
```

　　​`ExtensionLoader.getExtensionLoader(ExtensionFactory.class)`​这个我们已经熟悉了，是获取`ExtensionFactory`​的扩展点加载器，那后面这个方法`getAdaptiveExtension`​什么作用呢？

##### AdaptiveExtensionFactory

　　这个方法的作用是，获取接口实现类中含有`@Adaptive`​注解的实现类。对于`ExtensionFactory`​接口来说，获取的是`AdaptiveExtensionFactory`​类实力。 `AdaptiveExtensionFactory`​类在初始化的时候，又会寻找`ExtensionFactory`​的所有扩展点，把支持的扩展点类对象放到list中，即把`SpiExtensionFactory`​和`SpringExtensionFactory`​对象放到`factories`​中。

```java
public AdaptiveExtensionFactory() {
    ExtensionLoader<ExtensionFactory> loader = ExtensionLoader.getExtensionLoader(ExtensionFactory.class);
    List<ExtensionFactory> list = new ArrayList<ExtensionFactory>();
    for (String name : loader.getSupportedExtensions()) {
        list.add(loader.getExtension(name));
    }
    factories = Collections.unmodifiableList(list);
}
复制代码
```

##### SpringExtensionFactory

　　看Dubbo是如何从Spring容器中获取对象的呢？

```java
public <T> T getExtension(Class<T> type, String name) {
    if (type.isInterface() && type.isAnnotationPresent(SPI.class)) {
        return null;
    }
    for (ApplicationContext context : CONTEXTS) {
        T bean = getOptionalBean(context, name, type);
        if (bean != null) {
            return bean;
        }
    }
    return null;
}
public static <T> T getOptionalBean(ListableBeanFactory beanFactory, String beanName, Class<T> beanType) throws BeansException {
    if (beanName == null) {
        return getOptionalBeanByType(beanFactory, beanType);
    }
    T bean = null;
    try {
        bean = beanFactory.getBean(beanName, beanType);
    } catch (NoSuchBeanDefinitionException e) {
    } catch (BeanNotOfRequiredTypeException e) {
        logger.warn(String.format("bean type not match, name: %s, expected type: %s, actual type: %s",
                beanName, beanType.getName(), e.getActualType().getName()));
    }
    return bean;
}
复制代码
```

　　可以看到，先按接口类型`ByType`​方式从`beanFactory`​中查找，找不到的话再按`ByName`​方式从`beanFactory`​中查找，再找不到则返回null。

##### SpiExtensionFactory

```java
public <T> T getExtension(Class<T> type, String name) {
    if (type.isInterface() && type.isAnnotationPresent(SPI.class)) {
        ExtensionLoader<T> loader = ExtensionLoader.getExtensionLoader(type);
        if (!loader.getSupportedExtensions().isEmpty()) {
            return loader.getAdaptiveExtension();
        }
    }
    return null;
}
复制代码
```

　　如果从spring的bean工厂里没有找到对象的话，就从spi工厂里查找对象，根据接口类型获取对应的扩展点加载器，再获取对应的`Adaptive`​代理对象。也就是从接口的实现类中寻找有没有被`Adaptive`​注解标注的类，如果没有的话，会自动生成一个`Adaptive`​类，比如前面说的`Mascot$Adaptive`​。

##### 生成`Adaptive`​​代理类

　　生成代理类的代码在`org.apache.dubbo.common.extension.ExtensionLoader#createAdaptiveExtensionClass`​方法中，

```java
private Class<?> createAdaptiveExtensionClass() {
    String code = new AdaptiveClassCodeGenerator(type, cachedDefaultName).generate();
    ClassLoader classLoader = findClassLoader();
    org.apache.dubbo.common.compiler.Compiler compiler =
            ExtensionLoader.getExtensionLoader(org.apache.dubbo.common.compiler.Compiler.class).getAdaptiveExtension();
    return compiler.compile(code, classLoader);
}
```

　　也就是先生成代理类的代码，再通过编译生成对应的代理类。

　　生成代理类的代码在`org.apache.dubbo.common.extension.AdaptiveClassCodeGenerator`​类中，方法比较多，但也比较简单，就是按模板生成对应的代码。

### 总结 （ExtensionLoader）

#### 主要属性

##### EXTENSION_LOADERS

　　​`ConcurrentMap`​​类型，存放扩展点接口与扩展点加载器的对应关系：

　　​`Protocol.class`​​ ->  `Protocol的ExtensionLoader实例对象`​​

　　​`ExtensionFactory.class`​​  ->  `ExtensionFactory的ExtensionLoader实例对象`​​

##### EXTENSION_INSTANCES

　　​`ConcurrentMap`​​类型，存放扩展点类与扩展点类实例的对应关系：

　　​`DubboProtocol.class`​​ ->  `DubboProtocol实例对象`​​

　　​`SpiExtensionFactory.class`​​  ->  `SpiExtensionFactory实例对象`​​

##### type

　　当前ExtensionLoader实例是哪个接口的扩展点加载器，如：`Protocol.class`​​

##### objectFactory

　　对象工厂，用来获取对象，获取的对象可能是DubboSPI机制产生的，也可能是从Spring容器或其他容器中获取的。

##### cachedInstances

　　扩展点名字与扩展点实例对应关系

　　​`dubbo`​​ ->  `Holder<DubboProtocol>`​​

##### cachedDefaultName

　　默认扩展点的名字，如`@SPI("dubbo")`​​的dubbo

##### cachedWrapperClasses

　　缓存扩展点的包装类

#### 主要方法

###### getExtension

　　getExtension(String name)：用来获取`name`​​对应的扩展点实例对象，获取一次后，会被缓存，下次获取直接从缓存里拿。

###### getAdaptiveExtension

　　getAdaptiveExtension()：获取一个自适应的扩展点实例

###### createExtension

　　在创建一个扩展点实例时，过程如下：

1. 查找缓存`cachedClasses`​​是否已经有已经缓存的的扩展点class。
2. 如果没有则解析接口文件（`META-INF/dubbo/internal/`​​、`META-INF/dubbo/`​​、`META-INF/services/`​​），找到对应的扩展点class。
3. 从解析的class中，找到对应的扩展点实现类
4. 找到这个实现类后，生成一个实例对象，并放入缓存中。
5. 根据实现类生成一个实例，把实现类和对应生成的实例放入缓存中，方便下次获取
6. 对生成的实例对象进行属性赋值操作
7. 判断是否存在Wrapper包装类
8. 如果存在Wrapper包装类，则进行实例化并进行属性赋值
9. 如果有多个Wrapper，则会对多个Wrapper进行一层一层包装并赋值
10. 如果是Lifecycle类型，则进行初始化操作
11. 返回最终的Wrapper实例对象

###### loadExtensionClasses

　　​`loadExtensionClasses()`​​方法加载当前接口类型对应的所有扩展点实现类的，返回值类型是Map类型。返回的这个map最终放入`cachedClasses`​​缓存起来，下次获取直接从缓存中获取。

　　Dubbo在加载扩展点时，有3种读取策略，分别为`DubboInternalLoadingStrategy`​​、`DubboLoadingStrategy`​​、`ServicesLoadingStrategy`​​。这3个类分别负责读取的文件目录为`META-INF/dubbo/internal/`​​、`META-INF/dubbo/`​​、`META-INF/services/`​​。

　　读取时到3个目录寻找接口全限定名文件，再调用loadResource方法进行加载。

###### loadResource

　　该方法是对文件内容的解析，一行一行的解析，把解析出的内容按"="进行分隔，"="前面是扩展点的名字，后面是对应的实现类，拿到实现类后，用ExtensionLoader类的类加载器来加载扩展点实现类。

　　再调用`loadClass`​​方法对扩展点实例进行详细的解析，把解析后的最终对象放到map中缓存起来。

###### loadClass

　　​`loadClass`​​方法处理过程如下：

1. 判断当前扩展点实现类，是否实现了扩展点接口，如果没有实现则抛出异常
2. 判断当前扩展点实现类上是否被`@Adaptive`​​注解标记。
3. 如果存在的话，说明该类是当前接口的默认自适应类，并把该类赋值给`cachedAdaptiveClass`​​，作为缓存。
4. 判断当前扩展点实现类是否为`Wrapper`​​类，如何是的话，则把该类添加到`cachedWrapperClasses`​​这个Set集合中，`Wrapper`​​类可能存在多个，有多个添加多个。
5. 判断当前扩展点实现类是否存在无参构造方法，没有则报错。
6. 判断是否存在name，如果没有name，查看是否存在`Extension`​​注解。
7. 存在`Extension`​​注解，获取注解的值作为name。
8. 如果没有`Extension`​​注解获取类名处理后作为name。
9. 如果没有name，则报错。
10. 如果有多个name，则判断当前扩展点实现类上是否存在`@Activate`​​注解。
11. 如果存在`@Activate`​​注解，则把该类添加到`cachedActivates`​​缓存中。
12. 如果有多个name，则把每个name和对应的实现类存到`extensionClasses`​​中，最终会被放到`cachedClasses`​​中缓存起来。

###### injectExtension

　　此方法为Dubbo的IOC，处理过程如下：

1. 查找当前实例类中的setter方法
2. 判断setter是否通过注解`DisableInject`​​禁用了依赖注入
3. 判断setter方法的参数类型，如果是基本类型则跳过
4. 通过set方法名，获取要对应的属性值
5. 调用objectFactory实例的`getExtension`​​方法，根据参数类型和属性来获取对象，这个对象可能来着spring容器或者DubboSPI机制得到的对象。
6. 调用方法对象的invoke方法来进行赋值操作。

###### createAdaptiveExtensionClass

　　Dubbo的自适应扩展点对象是通过Dubbo内部生成代理类，通过编译产生的代理对象。

　　Dubbo可以通过`@Adaptive`​​注解类为某个接口生成代理类。Dubbo通过内部的代码模板生成一个类，然后编译产生一个代理对象。

　　使用`@Adaptive`​​注解产生代理类是有一定条件的。被`@Adaptive`​​注解的方法，必须有URL参数或者参数类有URL属性。

### 疑问

1. Wrapper类可以指定顺序吗？

　　Wrapper类是不支持指定顺序的，不是说谁写在前面谁就在里面，Dubbo内部会对Wrapper类进行排序的，在`org.apache.dubbo.common.extension.ExtensionLoader#createExtension`​​​方法中有体现。

　　如果有`@Activate`​​​注解，可以通过注解的`order`​​​属性执行顺序。

```java
List<Class<?>> wrapperClassesList = new ArrayList<>();
if (cachedWrapperClasses != null) {
    wrapperClassesList.addAll(cachedWrapperClasses);
    wrapperClassesList.sort(WrapperComparator.COMPARATOR);
    Collections.reverse(wrapperClassesList);
}
```

2. Dubbo的依赖注入会出现循环依赖吗？

　　Dubbo的依赖注入是不会出现循环依赖的，因为Dubbo在注入扩展点属性的时候，赋值的是一个`Adaptive`​​​代理类，而不是我们写的普通类。

3. 被`@Adaptive`​标注的方法的参数一定是`URL`​吗？

　　这个是不一定的，也可以是一个普通类，但普通类里面必须有`URL`​类型的属性，比如`Invoker`​这个类。

## 3.0

　　‍

　　‍

　　‍
