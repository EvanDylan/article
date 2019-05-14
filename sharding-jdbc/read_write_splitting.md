# 读写分离

## 背景

> 面对日益增加的系统访问量，数据库的吞吐量面临着巨大瓶颈。 对于同一时刻有大量并发读操作和较少写操作类型的应用系统来说，将数据库拆分为主库和从库，主库负责处理事务性的增删改操作，从库负责处理查询操作，能够有效的避免由数据更新导致的行锁，使得整个系统的查询性能得到极大的改善。
>
> 通过一主多从的配置方式，可以将查询请求均匀的分散到多个数据副本，能够进一步的提升系统的处理能力。 使用多主多从的方式，不但能够提升系统的吞吐量，还能够提升系统的可用性，可以达到在任何一个数据库宕机，甚至磁盘物理损坏的情况下仍然不影响系统的正常运行。
>
> 与将数据根据分片键打散至各个数据节点的水平分片不同，读写分离则是根据SQL语义的分析，将读操作和写操作分别路由至主库与从库。

## 核心功能

- 提供一主多从的读写分离配置，可独立使用，也可配合分库分表使用。
- 独立使用读写分离支持SQL透传。
- 同一线程且同一数据库连接内，如有写入操作，以后的读操作均从主库读取，用于保证数据一致性。
- 基于Hint的强制主库路由。

## 不支持项

- 主库和从库的数据同步。
- 主库和从库的数据同步延迟导致的数据不一致。
- 主库双写或多写。

## 实现原理

sharding-jdbc核心功能以下几个引擎所构成：解析引擎、路由引擎、改写引擎、执行引擎、归并引擎。各个引擎大致分工如下：

- 解析引擎：解析SQL语句，负责SQL语法树的构建，为后续的改写提供前置条件。
- 路由引擎：根据配置的分片字段与规则，解析具体的路由规则。
- 改写引擎：根据前两个引擎执行结果，对逻辑SQL进行改写生成实际需要执行的一条或者多条SQL语句。
- 执行引擎：负责执行由改写引擎重写生成的SQL依据。
- 归并引擎：负责对执行引擎的执行结果集进行汇总，做进一步的处理工作。

根据以上几个引擎的功能特点，可以推测出读写分离的功能主要在路由和改写两个步骤实现的。下面我们以读和写的两种业务场景分别展开进行分析实现过程。

### 读从库的实现

先来一张时序图，我们先从全局的视角体会下一个查询SQL执行的大致流程。

![](./images/05_01.jpg)

图中被标注出来的部分是我们下面将要分析的重点部分。

代码执行入口为`ShardingPreparedStatement`中`execute()`，方法内容如下：

```java
@Override
public boolean execute() throws SQLException {
    try {
        clearPrevious();
        // 负责SQL解析、改写、路由等
        shard();
        initPreparedStatementExecutor();
        // 对改写的SQL进行统一执行
        return preparedStatementExecutor.execute();
    } finally {
        clearBatch();
    }
}
```

进一步追踪`shard()`方法，会进入到`PreparedQueryShardingEngine`的`shard()`，方法内容如下：

```java
public SQLRouteResult shard(final String sql, final List<Object> parameters) {
    List<Object> clonedParameters = cloneParameters(parameters);
    SQLRouteResult result = route(sql, clonedParameters);
    result.getRouteUnits().addAll(HintManager.isDatabaseShardingOnly() ? convert(sql, clonedParameters, result) : rewriteAndConvert(sql, clonedParameters, result));
    if (shardingProperties.getValue(ShardingPropertiesConstant.SQL_SHOW)) {
        boolean showSimple = shardingProperties.getValue(ShardingPropertiesConstant.SQL_SIMPLE);
        SQLLogger.logSQL(sql, showSimple, result.getSqlStatement(), result.getRouteUnits());
    }
    return result;
}
```

## 写主库的实现

