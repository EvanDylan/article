dubbo优秀的扩展性表现在诸如注册中心、通讯协议、序列化协议等组件的插件多样性以及二次扩展开发的便捷性上。dubbo大部分的组件都以插件的方式提供，插件式架构也被称为微内核架构。在继续研究各个功能插件的细节之前，剖析下dubbo内核扩展机制显得尤为重要。

正式开始之前假设读者群体已经了解知道了Java中`SPI`机制，不了解的同学请先阅读[Java SPI介绍使用](./java_spi.md)。虽然Java语言自带的`SPI`也拥有不错的扩展性，但是在以下几个方面表现就有所欠缺：

- 启动时即加载所有扩展插件的实例对象，这对拥有大量扩展插件框架来说在使用到时再去加载显然会更加合适。（没有绝对的事情，如实应用追求运行时绝对的性能，那么懒加载可能并不是一个好选择，当然也可以通过启动预热等措施解决，这又是另外的话题了。总而言之，核心开发者经过取舍之后这就这么设计了）
- 插件对应用环境没有感知能力，当应用程序同时拥有多个扩展插件时，原生`SPI`无法根据上下文环境动态适配合适的插件。
- 另外dubbo的`SPI`也提供了有限的自动装配及代理功能（稍后会进行剖析）。

那么dubbo在重新实现自定义的`SPI`是如何做到的呢，下面我们就以`ExtensionLoader`类加载扩展插件为入口，一步步分析dubbo实现的具体细节。

源码中比较典型用法示例如下所示：

```java
public static Serialization getSerialization(URL url) {
    return ExtensionLoader.getExtensionLoader(Serialization.class).getExtension(
        url.getParameter(Constants.SERIALIZATION_KEY, Constants.DEFAULT_REMOTING_SERIALIZATION));
}
```

第一步传入要被加载实现类的接口类，其实就是给每个类型的接口分配了单独的`ExtensionLoader`。

```java
public static <T> ExtensionLoader<T> getExtensionLoader(Class<T> type) {
    // 省略无关代码
    ExtensionLoader<T> loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
    if (loader == null) {
        EXTENSION_LOADERS.putIfAbsent(type, new ExtensionLoader<T>(type));
        loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
    }
    return loader;
}
```

第二步传入要被加载实现类的别名，返回实现类的实例对象。

```java
public T getExtension(String name) {
    if (StringUtils.isEmpty(name)) {
        throw new IllegalArgumentException("Extension name == null");
    }
    // 如果指定扩展类别名是"true"，则加载默认的扩展类返回
    if ("true".equals(name)) {
        return getDefaultExtension();
    }
    // 获取之前缓存的对象，如果没有则创建一个新的
    final Holder<Object> holder = getOrCreateHolder(name);
    Object instance = holder.get();
    if (instance == null) {
        synchronized (holder) {
            instance = holder.get();
            if (instance == null) {
                // 创建指定类的实例对象
                instance = createExtension(name);
                holder.set(instance);
            }
        }
    }
    return (T) instance;
}
```

以上代码步骤拆开来看：

- 如果符合默认插件规则，优先返回默认对象。
- 从缓存中获取匹配的类实例对象，如果不存在则通过`createExtension`加载放入缓存然后返回。

进一步剖析`createExtension`方法

```java
private T createExtension(String name) {
    // 加载配置文件中的类
    Class<?> clazz = getExtensionClasses().get(name);
    if (clazz == null) {
        throw findException(name);
    }
    try {
        // 获取缓存对象，不存在则创建
        T instance = (T) EXTENSION_INSTANCES.get(clazz);
        if (instance == null) {
            EXTENSION_INSTANCES.putIfAbsent(clazz, clazz.newInstance());
            instance = (T) EXTENSION_INSTANCES.get(clazz);
        }
        // setter注入
        injectExtension(instance);
        // 初始构造参数为当前类的wrapperClass，替换为wrapperClass的实例对象
        Set<Class<?>> wrapperClasses = cachedWrapperClasses;
        if (CollectionUtils.isNotEmpty(wrapperClasses)) {
            for (Class<?> wrapperClass : wrapperClasses) {
                instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
            }
        }
        // 如果实现Lifecycle接口，则调用初始化方法
        initExtension(instance);
        return instance;
    } catch (Throwable t) {
        throw new IllegalStateException("Extension instance (name: " + name + ", class: " +
                                        type + ") couldn't be instantiated: " + t.getMessage(), t);
    }
}
```

以上代码主要涉及一下几个步骤：

- 首先确保类文件已经从配置文件中加载
- 获取已经加载过的缓存对象，如果没有则通过默认构造方法创建一个并加入缓存。
- 遍历获取到的对象所有方法，过滤出满足条件的`set`方法，并剔除声明了`DisableInject`注解的方法。
- 检查`cachedWrapperClasses`集合中所有以当前类作为构造方法参数的类，初始化并进行`set`注入。
- 如果当前对象还实现了`Lifecycle`，则调用`initi(alize()`方法。

# 读取配置文件

dubbo中约定的配置文件存放的路径

- `META-INF/dubbo/internal/`
- `META-INF/dubbo/`
- `META-INF/services/`

dubbo按照从上到下的顺序依次读取目录下含有指定类全限定名的文件，文件内容格式如`key=value`,`key`用来表示插件在dubbo中的名称，`value`则表示插件类的全限定名，将类载入放入到`cachedClasses`缓存中。

# 自动注入及动态代理实现

## set注入实现

着重分析`injectExtension(instance);`方法内容：

```java
private T injectExtension(T instance) {
    if (objectFactory == null) {
        return instance;
    }
    for (Method method : instance.getClass().getMethods()) {
        // 方法必须以set开头、必须是public、方法参数必须是一个
        if (!isSetter(method)) {
            continue;
        }
        // 方法如果有DisableInject注解，则跳过不处理
        if (method.getAnnotation(DisableInject.class) != null) {
            continue;
        }
        // 方法参数如果基础或者包装数据类型跳过
        Class<?> pt = method.getParameterTypes()[0];
        if (ReflectUtils.isPrimitives(pt)) {
            continue;
        }

        try {
            // 截取set方法名set后的字符串
            String property = getSetterProperty(method);
            // 获取set方法依赖的对象
            Object object = objectFactory.getExtension(pt, property);
            if (object != null) {
                // 反射调用将依赖对象注入
                method.invoke(instance, object);
            }
        } catch (Exception e) {
            logger.error("Failed to inject via method " + method.getName()
                         + " of interface " + type.getName() + ": " + e.getMessage(), e);
        }

    }
    return instance;
}
```

大体思路就是判断对象是否为`set`方法，如果满足一定的条件则通过`objectFactory`(下面介绍，先理解为Spring中的`beanFactory`)获取依赖对象，在通过反射调用`set`方法，完成`set`装配的功能。

##  构造方法注入及代理实现

在上面`set`注入完成之后开始进行构造方法注入的工作：

```java
Set<Class<?>> wrapperClasses = cachedWrapperClasses;
if (CollectionUtils.isNotEmpty(wrapperClasses)) {
    for (Class<?> wrapperClass : wrapperClasses) {
        instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
    }
}
```

`cachedWrapperClasses`这个对象是在加载类的时候创建的，判断扩展类是否`WrapperClass`逻辑如下：

```java
private boolean isWrapperClass(Class<?> clazz) {
    try {
        clazz.getConstructor(type);
        return true;
    } catch (NoSuchMethodException e) {
        return false;
    }
}
```

以dubbo中`RegistryFactory`、`ZookeeperRegistryFactory`、`RegistryFactoryWrapper`为例进一步说明两者之间关系。

```java
public class RegistryFactoryWrapper implements RegistryFactory {
    private RegistryFactory registryFactory;

    public RegistryFactoryWrapper(RegistryFactory registryFactory) {
        this.registryFactory = registryFactory;
    }

    @Override
    public Registry getRegistry(URL url) {
        return new ListenerRegistryWrapper(registryFactory.getRegistry(url),
                Collections.unmodifiableList(ExtensionLoader.getExtensionLoader(RegistryServiceListener.class)
                        .getActivateExtension(url, "registry.listeners")));
    }
}
```

当我们使用`zookeeper`作为注册中心时，会发生以下步骤：

1. 通过`ExtensionLoader.getExtensionLoader(RegistryFactory.class)`获取插件实例对象。
2. `ExtensionLoader`分别读取`dubbo-registry-zookeeper`模块及`dubbo-registry-api`模块下文件名为`org.apache.dubbo.registry.RegistryFactory`的配置。
3. 这里`RegistryFactoryWrapper`构造方法参数为`RegistryFactory`，所以`RegistryFactoryWrapper`会被加载`cachedWrapperClasses`缓存中。
4. 当创建`ZookeeperRegistryFactory`对象之后，会遍历`cachedWrapperClasses`，将对象通过构造方法注入。

这么做好处显而易见，可以通过`RegistryFactoryWrapper`完成代理的功能。

# 插件自动适配实现

典型的用法示例：

```java
RegistryFactory registryFactory = ExtensionLoader.getExtensionLoader(RegistryFactory.class).getAdaptiveExtension();
```

可以看到与上面用法的区别在于一个是`getExtension(String name)`、`getAdaptiveExtension()`，前者是显示指定需要加载的插件的名称，后者通过方法名称中"**adaptive**"一词的中文含义"**适应性**"也能推断出该方法自适应的特性。

dubbo的插件自动适配功能需要搭配注解`Adaptive`使用，在dubbo中该注解表现行为有两种：

- 如果该注解加在类上dubbo不会为该类生成代理类，代理功能由被注解类自己实现（参见`AdaptiveExtensionFactory`、`AdaptiveCompiler`）。
- 如果该注解加在方法上则dubbo会为该方法生成代理方法。

跟进`getAdaptiveExtension()`方法，通过双重检查锁调用`createAdaptiveExtension()`方法

```java
public T getAdaptiveExtension() {
    Object instance = cachedAdaptiveInstance.get();
    if (instance == null) {
        if (createAdaptiveInstanceError != null) {
            throw new IllegalStateException("Failed to create adaptive instance: " +
                    createAdaptiveInstanceError.toString(),
                    createAdaptiveInstanceError);
        }

        synchronized (cachedAdaptiveInstance) {
            instance = cachedAdaptiveInstance.get();
            if (instance == null) {
                try {
                    // 创建自适应插件
                    instance = createAdaptiveExtension();
                    cachedAdaptiveInstance.set(instance);
                } catch (Throwable t) {
                    createAdaptiveInstanceError = t;
                    throw new IllegalStateException("Failed to create adaptive instance: " + t.toString(), t);
                }
            }
        }
    }

    return (T) instance;
}
```

继续跟进`createAdaptiveExtension()`方法

```java
private T createAdaptiveExtension() {
    try {
        return injectExtension((T) getAdaptiveExtensionClass().newInstance());
    } catch (Exception e) {
        throw new IllegalStateException("Can't create adaptive extension " + type + ", cause: " + e.getMessage(), e);
    }
}
```

核心代码拆开来看分为以下步骤：

- 通过`getAdaptiveExtensionClass`获取插件类。
- 通过调用`newInstance()`创建对象实例。
- 通过调用`injectExtension`完成set注入功能。

继续跟进`getAdaptiveExtensionClass`

```java
private Class<?> getAdaptiveExtensionClass() {
    // 加载配置文件中的类
    getExtensionClasses();
    // 在加载类过程中，如果插件类被注解Adaptive则被赋值给当前对象。
    if (cachedAdaptiveClass != null) {
        // 返回之后直接通过createAdaptiveExtension方法中的newInstance完成对象实例的创建
        return cachedAdaptiveClass;
    }
    // 如果Adaptive注解在方法上走这个分支逻辑
    return cachedAdaptiveClass = createAdaptiveExtensionClass();
}
```

## 注解在类上

通过`getAdaptiveExtensionClass`可知如果类被`Adaptive`注解标注，则直接返回该类完成对象实例创建过程。应用到代码中`AdaptiveExtensionFactory`、`AdaptiveCompiler`这两个类直接调用默认的无参构造方法即为完成了初始化的过程。

## 注解在方法上

继续跟进`createAdaptiveExtensionClass`方法：

```java
private Class<?>  createAdaptiveExtensionClass() {
    // 动态生成类的文本内容
    String code = new AdaptiveClassCodeGenerator(type, cachedDefaultName).generate();
    // 将文本编译为类对象
    ClassLoader classLoader = findClassLoader();
    org.apache.dubbo.common.compiler.Compiler compiler = 		           ExtensionLoader.getExtensionLoader(org.apache.dubbo.common.compiler.Compiler.class).getAdaptiveExtension();
    return compiler.compile(code, classLoader);
}
```

核心内容在于动态生成类，在展开分析具体过程之前，先来看个简单的示例，现有如下用法：

```java
RouterFactory routerFactory = ExtensionLoader.getExtensionLoader(RouterFactory.class).getAdaptiveExtension();
```

其中`RouterFactory`内容如下：

```java
@SPI
public interface RouterFactory {

    /**
     * Create router.
     * Since 2.7.0, most of the time, we will not use @Adaptive feature, so it's kept only for compatibility.
     *
     * @param url url
     * @return router instance
     */
    @Adaptive("protocol")
    Router getRouter(URL url);
}
```

生成的代理内容简化如下：

```java
public class RouterFactory$Adaptive implements RouterFactory {
    public Router getRouter(URL arg0) {
        if (arg0 == null) throw new IllegalArgumentException("url == null");
        URL url = arg0;
        String extName = url.getProtocol();
        if (extName == null)
            throw new IllegalStateException("Failed to get extension (RouterFactory) name from url (" + url.toString() + ") use keys([protocol])");
        RouterFactory extension = (RouterFactory) ExtensionLoader.getExtensionLoader(RouterFactory.class).getExtension(extName);
        return extension.getRouter(arg0);
    }
}
```

可以初步了解到代理类，实现了`RouterFactory`中的`getRouter`方法。`@Adaptive("protocol")`中的"**protocol**"属性被解析成`url.getProtocol()`方法的一部分，通过这种方式动态获取应用配置的插件类型，继而通过`getExtension(extName)`获取真正的插件实现类。

下面开始继续分析生成代理类的过程：

```java
public String generate() {
    // no need to generate adaptive class since there's no adaptive method found.
    if (!hasAdaptiveMethod()) {
        throw new IllegalStateException("No adaptive method exist on extension " + type.getName() + ", refuse to create the adaptive class!");
    }

    StringBuilder code = new StringBuilder();
    // 生成包名
    code.append(generatePackageInfo());
    // import ExtensionLoader
    code.append(generateImports());
    // 生成类签名，形如 public class %s$Adaptive implements %s {
    code.append(generateClassDeclaration());
	// 生成方法
    Method[] methods = type.getMethods();
    for (Method method : methods) {
        code.append(generateMethod(method));
    }
    code.append("}");

    if (logger.isDebugEnabled()) {
        logger.debug(code.toString());
    }
    return code.toString();
}
```

一行行分析`RouterFactory$Adaptive`过程。

`generatePackageInfo()`获取`RouterFactory`包名，拼接之后结果为`package org.apache.dubbo.rpc.cluster`

`generateClassDeclaration()`获取`ExtensionLoader`类型，拼接之后结果为`import org.apache.dubbo.common.extension. ExtensionLoader`

`generateClassDeclaration()`，拼接之后结果为`public class RouterFactory$Adaptive implements org.apache.dubbo.rpc.cluster.RouterFactory {`

继续跟进`generateMethod(method)`，因为`RouterFactory`只有一个方法`Router getRouter(URL url)`，下面分析以该方法作为示例

```java
private String generateMethod(Method method) {
    String methodReturnType = method.getReturnType().getCanonicalName();
    String methodName = method.getName();
    String methodContent = generateMethodContent(method);
    String methodArgs = generateMethodArguments(method);
    String methodThrows = generateMethodThrows(method);
    return String.format(CODE_METHOD_DECLARATION, methodReturnType, methodName, methodArgs, methodThrows, methodContent);
}
```

`method.getReturnType().getCanonicalName()`结果为`org.apache.dubbo.rpc.cluster.Router`

`method.getName()`结果为`getRouter`

`generateMethodArguments(method)`结果为`org.apache.dubbo.common.URL arg0`

`generateMethodThrows(method)`，因为方法没有抛出异常，所以这里是空的。

继续跟进`generateMethodContent`方法

```java
private String generateMethodContent(Method method) {
    Adaptive adaptiveAnnotation = method.getAnnotation(Adaptive.class);
    StringBuilder code = new StringBuilder(512);
    if (adaptiveAnnotation == null) {
        return generateUnsupported(method);
    } else {
        // 找到方法参数为URL是第几个参数,这里显示是0
        int urlTypeIndex = getUrlTypeIndex(method);

        // 如果找到了URL参数
        if (urlTypeIndex != -1) {
            // Null Point check
            // 生成 if (arg0 == null) throw new IllegalArgumentException("url == null"); org.apache.dubbo.common.URL url = arg0;
            code.append(generateUrlNullCheck(urlTypeIndex));
        } else {
            // did not find parameter in URL type
            // 如果没有找到URL参数，则循环该方法参数的所有方法是有满足以下条件的方法：
            // 以get开头、public修饰符、非静态、入参为0、返回值类型为URL的
            code.append(generateUrlAssignmentIndirectly(method));
        }
		// 返回"protocol"
        String[] value = getMethodAdaptiveValue(adaptiveAnnotation);
		// 是否含有Invocation参数
        boolean hasInvocation = hasInvocationArgument(method);
         // 这里getRouter(URL url)没有Invocation参数，所有返回值是空
		// if (arg%d == null) throw new IllegalArgumentException(\"invocation == null\"); String methodName = arg%d.getMethodName();
        code.append(generateInvocationArgumentNullCheck(method));
		// 这里分支比较多，最后会生成 String extName = url.getProtocol();
        code.append(generateExtNameAssignment(value, hasInvocation));
        // 非空校验 if (extName == null) throw new IllegalStateException("Failed to get extension (org.apache.dubbo.rpc.cluster.RouterFactory) name from url (" + url.toString() + ") use keys([protocol])");
        code.append(generateExtNameNullCheck(value));
		// 生成获取插件对象代码 org.apache.dubbo.rpc.cluster.RouterFactory extension = (org.apache.dubbo.rpc.cluster.RouterFactory) ExtensionLoader.getExtensionLoader(org.apache.dubbo.rpc.cluster.RouterFactory.class).getExtension(extName);
        code.append(generateExtensionAssignment());

        // 生成调用代理对象getRouter方法 return extension.getRouter(arg0);
        code.append(generateReturnAndInvocation(method));
    }

    return code.toString();
}
```

导致完整代理类内容就生成了：

```java
package org.apache.dubbo.rpc.cluster
import org.apache.dubbo.common.extension. ExtensionLoader
    
public class RouterFactory$Adaptive implements org.apache.dubbo.rpc.cluster.RouterFactory {
    public org.apache.dubbo.rpc.cluster.Router getRouter(org.apache.dubbo.common.URL arg0) {
        if (arg0 == null) throw new IllegalArgumentException("url == null");
        org.apache.dubbo.common.URL url = arg0;
        String extName = url.getProtocol();
        if (extName == null) throw new IllegalStateException("Failed to get extension (org.apache.dubbo.rpc.cluster.RouterFactory) name from url (" + url.toString() + ") use keys([protocol])");
        org.apache.dubbo.rpc.cluster.RouterFactory extension = (org.apache.dubbo.rpc.cluster.RouterFactory) ExtensionLoader.getExtensionLoader(org.apache.dubbo.rpc.cluster.RouterFactory.class).getExtension(extName);
        return extension.getRouter(arg0);
    }
}
```

这部分代码比较琐碎，毕竟要编码完成生成代码以及编译等工作，可以作为设计的一种思路，生成的代码的源码部分实在是没有仔细研读的必要。

通过本篇部分，完成了对dubbo自定义`SPI`实现原理及源码的解析，希望可以给大家带来一点收获。