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

## 缓存的对与错

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

可以看到在同一个查询在SqlSession生命周期内只被执行了一次，返回的对象地址也是完全一致的，可以证明一级缓存的存在。一级缓存看似很美好，但是同样了带了其他的副作用。第一点是：同一个事物内的更新



## 执行流程

在请求过程中缓存作用流程

![](./images/05_10.png)

