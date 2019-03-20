# 一二级缓存

## 作用域与生命周期

比较重要的对象的作用域和生命周期：

|     实例对象      |   作用域   |                           生命周期                           |
| :---------------: | :--------: | :----------------------------------------------------------: |
| SqlSessionFactory | 应用作用域 |                  应用启动创建，应用停止销毁                  |
|    SqlSession     | 请求作用域 | 非线程安全，保持在请求的ThreadLocal中，随着请求而创建、请求结束销毁 |
|      Mapper       | 请求作用域 |                       同SqlSession一样                       |

一二级缓存的作用域和生命周期

|          |            作用域             |                     生命周期                     |                启动方式                |
| :------: | :---------------------------: | :----------------------------------------------: | :------------------------------------: |
| 一级缓存 |        SqlSession级别         | 伴随SqlSession创建而生，随着SqlSession关闭而消亡 |                默认开启                |
| 二级缓存 | 以Mapper对应的namesapce为纬度 |              伴随着整个应用生命周期              | 需要在对应的mapper下配置<cache/>来开启 |

## 一级缓存

### 利与弊

我们先来验证下一级缓存的存在，执行以下单元测试

```java
@Test
public void queryUserTest() {
    UserMapper userMapper = sqlSession.getMapper(UserMapper.class);
    User user1 = userMapper.queryUser(1L);
    User user2 = userMapper.queryUser(1L);
    System.out.println(user1 == user2);
}
```

控制台结果:

![](./images/05_01.png)

可以看到在同一个查询在SqlSession生命周期内只被执行了一次，返回的对象地址也是完全一致的，可以证明一级缓存的存在。在一定程度上一级缓存确实可以提升应用的查询性能，一切看似很美好，但是如果使用不当也会出现问题。

请考虑以下应用场景：在一个SqlSession生命周期内的同一个SQL前后多次执行时，如果在多次执行的期间有其他的线程对数据进行变更，但是由于缓存的存在，那么多次读取都会始终返回第一次查询的结果，在这种情况下就会产生缓存和数据库数据一致的情况。

但是实际业务场景下会有人这样写代码么，即便存在特殊的业务场景必须要这么处理，但毕竟还是少数。所以综合考量我认为Mybatis开发者这么去架构设计是综合考量的结果（利 > 弊），同时针对这种特殊的业务场景也提供了关闭一级缓存的选项。比如本例子中的`user-mapper.xml`就可以将查询语句中的`flushCache`由默认的`false`更改为`true`：

```xml
<mapper namespace="org.apache.ibatis.UserMapper">
    <select id="queryUser" resultType="org.apache.ibatis.User" flushCache="true">
        select * from user where id = #{id}
    </select>
</mapper>
```

更新完配置之后再次执行单元测试，控制台输出结果如下：

![](./images/05_03.png)

可以看出同一个SqlSession内的第二次查询不再从缓存中获取，而是直连db查询。

### 深究实现原理

上面我们讨论完了一级缓存的利与弊，下面我们继续探究一级缓存是在何时写入，它的更新机制以及更新机制所带来的问题。

先来认识下Mybatis中一级缓存的实现类`PerpetualCache`类，内容如下：

```java
public class PerpetualCache implements Cache {

    private final String id;

    private Map<Object, Object> cache = new HashMap<>();

    public PerpetualCache(String id) {
      this.id = id;
    }

    @Override
    public String getId() {
      return id;
    }

    @Override
    public int getSize() {
      return cache.size();
    }

    @Override
    public void putObject(Object key, Object value) {
      cache.put(key, value);
    }

    @Override
    public Object getObject(Object key) {
      return cache.get(key);
    }

    @Override
    public Object removeObject(Object key) {
      return cache.remove(key);
    }

    @Override
    public void clear() {
      cache.clear();
    }
    // 省略非主要代码
}
```

这里可以看到默认的一级缓存使用非线程安全集合类`HashMap`来实现。为什么不会出现线程安全的问题呢？且听我慢慢道来。`PerpetualCache`这里跟`BaseExecutor`有着千丝万缕的关系（`BaseExecutor`又是什么，没关系请继续往下看）。

在架构总览的时候有提到`Executor`是执行SQL的执行器，它的相关的类家族UML图如下所示，`SimpleExecutor`是默认使用的执行器；`ReuseExecutor`将SQL与Statement缓存起来，这样就不用重复的创建新的Statement；`BatchExecutor`是批量执行器，通过封装底层JDBC的BATCH相关的API，来加速批量添加、删除等业务场景。而我们关系的一级缓存`PerpetualCache`则以全局变量存在于`BaseExecutor`中。

![](./images/05_04.png)

那么执行器是在什么时候创建的呢？我们先来看下创建的时序图。

![](./images/05_05.png)

在时序图中我们可以看到`Executor`在获取`SqlSession`时每次都会被重新创建，在开篇的时候有提到`SqlSession`随着请求而创建，伴随着请求结束而消亡，作用域也是限于当前线程的上下文。所以在并发请求的业务场景，`Executor`也是不同的，一级缓存也不会是多线程共享的，故而不会出现并发问题（但是你如果非要调皮在单个请求里创建子线程，去玩些骚操作那也没得办法）。这里我们的一级缓存容器已经创建完毕，那么缓存是如何被使用、存入以及更新的呢？且看下面的分析。

缓存被使用肯定是在查询的环节，我们先看下整体的粗略的查询流程的时序图：

![](./images/05_06.png)

图中我们可以看到最终会调用`BaseExecutor`中的`query`方法，`query`方法的内容如下：

```java
public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    ErrorContext.instance().resource(ms.getResource()).activity("executing a query").object(ms.getId());
    if (closed) {
      throw new ExecutorException("Executor was closed.");
    }
    // 如果不是嵌套查询或者嵌套查询执行完毕，并且flushCache配置为true
    if (queryStack == 0 && ms.isFlushCacheRequired()) {
      // 清空一级缓存
      clearLocalCache();
    }
    List<E> list;
    try {
      // 本次查询执行查询次数+1
      queryStack++;
      list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
      if (list != null) {
        handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
      } else {
        // 没有命中缓存则到数据库中查询
        list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
      }
    } finally {
      // 本次查询执行查询次数-1
      queryStack--;
    }
    // 查询执行完毕
    if (queryStack == 0) {
      for (DeferredLoad deferredLoad : deferredLoads) {
        deferredLoad.load();
      }
      deferredLoads.clear();
      // 如果本地缓存Scope被配置为STATEMENT，则每次执行的SQL的时候都要清空缓存。
      // 因为有queryStack == 0的判断，所以查询如果是嵌套查询，缓存还是起作用的。
      if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
        clearLocalCache();
      }
    }
    return list;
  }
```

1.  在源码`if (queryStack == 0 && ms.isFlushCacheRequired())`这一行，如果条件为true时会清空本地的缓存。这个正是我们上面在mapper中查询的SQL中使用到的`flushCache = true`参数。

2. 后续的则是正常的判断缓存是否存在，不存则从数据库中查询的套路。

3. 源码的`if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT)`这一行关系到了另外一个比较重要的参数`localCacheScope`。从代码中可知，如果设置为`STATEMENT`那么每次执行查询操作的时候都会清空一级缓存（嵌套查询除外）。对照下官方的解释：“若设置值为 STATEMENT，本地会话仅用在语句执行上，对相同 SqlSession 的不同调用将不会共享数据。”

   那么怎么使用呢？这个参数可选项有`SESSION`、`STATEMENT`两种，官方默认值为`SESSION`，可以通过如下配置更改默认值。

   ```
   <configuration>
       <settings>
           <setting name="localCacheScope" value="STATEMENT"/>
       </settings>
   </configuration>
   ```



## 执行流程

在请求过程中缓存作用流程

![](./images/05_10.png)

