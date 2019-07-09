# Bean的生命周期及作用域

## 生命周期

### 时序图

从使用角度出发，Spring有两种最为常用的装配Bean的方式：通过配置文件声明各个Bean创建过程以及各个Bean之间的依赖关系；或者通过更加便捷的方式注解来达到同样的目的。这两种初始化的方式实现类的分别为：`AnnotationConfigApplicationContext`、`ClassPathXmlApplicationContext`。

两者初始化时序图如下所示：

注解方式

![](../images/01_01.jpg)

配置文件方式：

![](../images/01_02.jpg)

时序图对于第一次接触的同学可能有点稍微复杂了些，但是其中的细节不必过于关注。对以上过程可以从宏观上总结如下：两种解析方式最终呈现的结果都是解析完成后的`BeanDefinition`，而且都会调用`AbstractApplicationContext`的`refresh()`方法来完成加载，那么这个方法就是我们下面要分析的重中之重，我们先来看下这个方法的内容：

```java
public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// 初始化刷新的上下文
			// Prepare this context for refreshing.
			prepareRefresh();

			// 将Bean的配置解析为BeanDefinition,并注册到BeanDefinitionRegistry中供后续真正初始化时使用
			// Tell the subclass to refresh the internal bean factory.
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// 准备BeanFactory上下文信息
			// Prepare the bean factory for use in this context.
			prepareBeanFactory(beanFactory);

			try {
				// Allows post-processing of the bean factory in context subclasses.
				postProcessBeanFactory(beanFactory);

				// 调用容器中所有实现了BeanFactoryPostProcessor接口实例的postProcessBeanFactory方法
				// Invoke factory processors registered as beans in the context.
				invokeBeanFactoryPostProcessors(beanFactory);

				// 将容器中所有实现了BeanPostProcessor接口的实例注册添加到到BeanFactory中beanPostProcessors属性中
				// Register bean processors that intercept bean creation.
				registerBeanPostProcessors(beanFactory);

				// 初始化MessageSource组件
				// Initialize message source for this context.
				initMessageSource();

				// 初始化事件广播组件
				// Initialize event multicaster for this context.
				initApplicationEventMulticaster();

				// 扩展方法,子类如果希望在refresh()方法执行的时候回调完成某些功能,可以重写onRefresh()方法
				// Initialize other special beans in specific context subclasses.
				onRefresh();

				// 将事件监听器添加到上面初始化过的广播组件中
				// Check for listener beans and register them.
				registerListeners();

				// 初始化所有剩余非延迟加载的实例
				// Instantiate all remaining (non-lazy-init) singletons.
				finishBeanFactoryInitialization(beanFactory);

				// 收尾工作
				// Last step: publish corresponding event.
				finishRefresh();
			}
			// 省略catach finally代码
		}
	}
```

将代码翻译为Bean生命周期的流程图如下：

![](../images/01_03.jpg)

### 示例

下面我们通过一个简单的示例来验证下我们上面的结论：

创建自定义的类`CustomBean`分别实现`BeanNameAware`、`BeanFactoryAware`、`ApplicationContextAware`、`InitializingBean`、`DisposableBean`接口，内容如下：

```java
public class CustomBean implements BeanNameAware, BeanFactoryAware, ApplicationContextAware,
		InitializingBean, DisposableBean {

	@Override
	public void setBeanName(String name) {
		System.out.println("BeanNameAware");
	}

	@Override
	public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
		System.out.println("BeanFactoryAware");
	}

	@Override
	public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
		System.out.println("ApplicationContextAware");
	}

	@Override
	public void destroy() throws Exception {
		System.out.println("DisposableBean destroy");
	}

	@Override
	public void afterPropertiesSet() throws Exception {
		System.out.println("InitializingBean afterPropertiesSet");
	}
}
```

创建类`CustomBeanFactoryPostProcessor`实现`BeanFactoryPostProcessor`接口，内容如下：

```java
public class CustomBeanFactoryPostProcessor implements BeanFactoryPostProcessor {
	@Override
	public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
		System.out.println("BeanFactoryPostProcessor");
	}
}
```

创建类`CustomBeanPostProcessor`实现`BeanPostProcessor`接口，内容如下：

```java
public class CustomBeanPostProcessor implements BeanPostProcessor {

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("BeanPostProcessor before");
        return bean;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException{
        System.out.println("BeanPostProcessor after");
        return bean;
    }
}
```

``

创建测试类：

```java
@Configuration
public class Main {

	@Bean
	public CustomBean customBean() {
		return new CustomBean();
	}

	@Bean
	public CustomBeanFactoryPostProcessor customBeanFactoryPostProcessor() {
		return new CustomBeanFactoryPostProcessor();
	}

	@Bean
	public CustomBeanPostProcessor customBeanPostProcessor() {
		return new CustomBeanPostProcessor();
	}

	public static void main(String[] args) {
		AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(Main.class);
		context.close();
	}
}
```

运行之后在控制台就能看到如下顺序的输出内容：

> BeanFactoryPostProcessor
> BeanNameAware
> BeanFactoryAware
> ApplicationContextAware
> BeanPostProcessor before
> InitializingBean afterPropertiesSet
> BeanPostProcessor after
> DisposableBean destroy

上面的示例简单的证实了Spring容器中Bean整个生命周期内的关键扩展点的作用顺序，熟悉这些扩展点有什么用呢？别着急，我们下面来看几个Spring内部进行的一些扩展，紧接着我们再介绍一些第三方框架的扩展点。

### 漫谈扩展点

#### BeanNameAware

作用：Spring通过回调该接口的`setBeanName(String name)`方法，将实现该接口Bean在容器中的名称传递。

调用时机：在Bean完成属性的装配之后，但是在还没完成实例初始化之前。

#### BeanFactoryAware

作用：Spring通过回调该接口的`setBeanFactory(BeanFactory beanFactory)`方法，将实现该接口的Bean的容器（BeanFactory）传递。

调用时机：仅挨着`BeanNameAware`，但是在还没完成实例初始化之前。

#### ApplicationContextAware

这个接口其实是借助于下面要介绍到`BeanPostProcessor `来实现的，感兴趣的可以看下`ApplicationContextAwareProcessor`的实现内容。

作用：同`BeanFactoryAware`类似，只是传递是`ApplicationContext`。

调用时机：同`BeanPostProcessor `一致。

#### InitializingBean 

#### DisposableBean 



## 作用域



