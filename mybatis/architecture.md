# 架构总览

## 核心功能

![](./images/03_01.png)

- 最底层的`Logging`服务启动时按照以下顺序`slf4j`、`commons logging`、`log4j2`、`log4j`、`jdk logging`，自动加载存在的日志系统。并通过适配器模式提供了统一的日志门面`Log`接口，并提供了适配不同日志系统的实现。Mybatis中通过`ErrorContext`类在流程的各个关键位置提供了错误追溯的功能，做到错误有迹可循。

- 初始化时通过`IO`包下的`Resources`类读取配置文件配合上`builder`包下的`XMLConfigBuilder`将外部的配置属性收口到一个类`Configuration`中。
- 外部定义的`Mapper`相关接口通过Mybatis中`Reflection`包提供的代理反射功能，将参数的解析，连接获取、sql执行、结果集映射、连接关闭、以及事务等等工作帮我们在内部隐藏消化。

## 关键步骤的时序图

###  SqlSession创建

![](./images/03_02.jpg)

###mapper代理类创建

![](./images/03_03.jpg)

### select执行

![](./images/03_04.jpg)

高清大图请点击[链接](https://www.processon.com/view/link/5c8b53a2e4b0f88919af568d)