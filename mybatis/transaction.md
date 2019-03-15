# 事务管理

在介绍事务管理之前，先简单介绍下Mybatis中对于数据库连接管理的方案。Mybatis中默认提供了三种解决，分别为：`JndiDataSourceFactory`、`PooledDataSourceFactory`、`UnpooledDataSourceFactory`，但是我们平时在开发时往往会更加青睐于选择功能更加完善、性能更加强劲的其他开源实现，譬如`druid`、`HikariCP`等。但是不管框架千变万化，连接池初衷始终是为了减少频繁的打开和关闭连接操作，减少IO的开销来提升应用的性能。

