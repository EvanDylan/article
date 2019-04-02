# 插件

## 概要

Mybatis允许在SQL执行过程某些点进行拦截。默认情况，Mybatis允许的调用拦截点有：

- Executor (update, query, flushStatements, commit, rollback, getTransaction, close, isClosed) 
- ParameterHandler (getParameterObject, setParameters)
- StatementHandler (prepare, parameterize, batch, update, query)
- ResultSetHandler (handleResultSets, handleOutputParameters)

`Executor `扮演者执行器的角色，是所有的SQL执行的总入口；`ParameterHandler`负责对SQL语句的输入参数进行处理；`StatementHandler `负责SQL语句执行调用、`Statement`的初始化、调用`ParameterHandler`将输入参数绑定到`Statement`上；`ResultSetHandler `对执行的结果集和实体进行映射处理。

可以看出Mybatis在SQL执行前、输入参数处理、SQL执行、结果集处理等这些重要的步骤中都预留了丰富的拦截点。即便我们平时用到的分页插件`PageHelper`也是基于这些基本的拦截点开发的。如果你现在已经迫不及待的想要想开发出自己的插件，但是需要特别注意官网强调的内容：

> The details of these classes methods can be discovered by looking at the full method signature of each, and the source code which is available with each MyBatis release. You should understand the behaviour of the method you’re overriding, assuming you’re doing something more than just monitoring calls. If you attempt to modify or override the behaviour of a given method, you’re likely to break the core of MyBatis. These are low level classes and methods, so use plug-ins with caution.

大概意思：如果你想做的不仅仅方法调用的监控，那么你最好需要了解重新方法的行为（翻译为“作用”可能会更好）。如果你尝试修改或者重写这些方法的内容，你很可能会破坏Mybatis的核心功能。因为它们都是比较底层的类和方法，所以使用插件的时候要特别的小心！今天我们不是要去探讨上面每个方法的作用，而是探究Mybatis这么强大的插件机制是如何实现。

## 简单的示例

在这个示例中我们将开发一个自己的监控的插件，用以监控每个SQL执行的真正时长。

1. 插件内容如下

   ```java
   @Intercepts({
           @Signature(type = Executor.class, method = "query", args = {MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class}),
           @Signature(type = Executor.class, method = "query", args = {MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class, CacheKey.class, BoundSql.class}),
           @Signature(type = Executor.class, method = "update", args = {MappedStatement.class, Object.class})})
   public class MonitorSQLExecutionTimePlugin implements Interceptor {
       public Object intercept(Invocation invocation) throws Throwable {
           Object[] args = invocation.getArgs();
           MappedStatement mappedStatement = (MappedStatement) args[0];
           Object param = args[1];
   
           BoundSql boundSql = mappedStatement.getBoundSql(param);
           String sql = boundSql.getSql();
   
           long beginTime = System.currentTimeMillis();
           Object result = invocation.proceed();
           long endTime = System.currentTimeMillis();
   
           System.out.println(MessageFormat.format("sql : {0}, elapsed time {1} ms", sql, endTime - beginTime));
           return result;
       }
   
       public Object plugin(Object target) {
           return Plugin.wrap(target, this);
       }
   
       @Override
       public void setProperties(Properties properties) {
   
       }
   }
   ```

2. 在全局配置文件中将刚刚开发完成的插件配置启用

   ```xml
   <plugins>
       <plugin interceptor="org.rhine.mybatis.demo.plugins.MonitorSQLExecutionTimePlugin"/>
   </plugins>
   ```

3. 在运行查询的时候，控制台输出如下的内容

   ![](./images/08_01.png)

   控制台输出了我们所期望的SQL语句及SQL的执行时间。

## 原理探究

