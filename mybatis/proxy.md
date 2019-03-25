# Mapper动态代理

写这篇的博文的目的是源于对“为什么项目中可以直接调用mapper接口？”这个问题的思考。第一感觉肯定是使用了动态代理，但是Mybatis底层是如何实现的？使用到了什么技术？

## JDK中的动态代理

在正式介绍MyBatis动态代理之前，先了解下JDK动态代理用法。

接口类`Engine`内容：

```java
interface Engine {

    void startEngine();

}
```

接口实现类`Car`:

```java
public class Car implements Engine {

    @Override
    public void startEngine() {
        System.out.println("start the engine");
    }
}
```

现在我们想在`startEngine`这个方法执行的时候进行代理，在方法执行前后织入我们自己的业务内容。需要实现`InvocationHandler`这个接口，内容如下：

```java
public class ProxyImplement implements InvocationHandler {

    private Engine engine;

    public ProxyImplement(Engine engine) {
        this.engine = engine;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("准备启动引擎...");
        Object result = method.invoke(engine, args);
        System.out.println("引擎启动完成...");
        return result;
    }
}
```

通过`Proxy`类的`newProxyInstance`动态生成代理类。

```java
public class ProxyFactory {

    public static <T> T getInstance(Engine engine, InvocationHandler invocationHandler) {
        return (T) Proxy.newProxyInstance(engine.getClass().getClassLoader(), engine.getClass().getInterfaces(), invocationHandler);
    }
}
```

测试代码如下：

```java
public class Main {

    public static void main(String[] args) {

        Engine engine = new Car();
        ProxyImplement proxyImplement = new ProxyImplement(engine);

        Engine proxyEngine = ProxyFactory.getInstance(engine, proxyImplement);
        proxyEngine.startEngine();
    }
}
```