# 一二级缓存

## Scope and Lifecycle

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

## 缓存的利与弊

1.  我们先来验证下一级缓存的存在，执行以下单元测试

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

上面我们讨论完了一级缓存的利与弊，下面我们继续探究一级缓存是在何时写入，以及它的更新机制。







## 执行流程

在请求过程中缓存作用流程

![](./images/05_10.png)

