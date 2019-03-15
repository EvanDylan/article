# 代码模块

- `annotations ` 注解相关。
- `cache` Mybatis提供了不同机制的缓存实现。
- `datasource` 提供了不同种类的数据源实现方案，比如jndi、带有连接池、不带连接池。
- `executor` Mybatis执行sql并对返回结果做映射。
- `jdbc` 提供了`SQL`类可以更加方便在代码中使用动态sql，并提供了`ScriptRunner`和`SqlRunner`可以直接执行sql脚本。
- `logging` 日志记录模块，提供了针对不同日志实现的适配方案，并且是自动发现应用，同时也支持手动指定。
- `mapping` 对参数以及返回结果的封装。
- `parsing` 提供Property文件以及XML文件的解析功能。
- `plugin` 拦截器功能。
- `reflection` 提供动态代理功能。
- `session` SqlSession相关实现。
- `transaction` 提供了非常弱的一层事务管理功能。
- `type` 提供数据库column到Java对象属性的转换功能。