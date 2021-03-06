## 大话分库分表

分库分表这一话题可谈的点实在太多，从技术产生的历史背景、实现原理、解决了哪些问题以及又带来了哪些新的问题等。由于才疏学浅，就不能庖丁解牛似的详细展开，只能泛泛而谈。

- 技术产生历史背景

- 解决了哪些问题

  传统RDBMS由于需要保证ACID的特性就决定了它不太可能实现分布式部署（技术的发展谁也说不准），在这种场景下必然存在这单机的性能瓶颈。后续发展出现的双主（解决单点问题）、主从（提高查询性能）、一主多从（提供查询性能）等等部署的架构也都是解决了特定场景下的问题。但始终没能彻底解决高并发写入的性能问题、超大表的查询性能问题。问题的根源在于：当性能瓶颈是单表时，而表在数据库层面又是不可切分的基本单位。正常的情况下一个表只会存储在固定的一台服务器上的某个磁盘节点上，由于单磁盘的限制其物理性能总是有限的，对比互联网行业的海量高并发其必然会在某个业务爆发期成为性能瓶颈。那么将超大的表进行合理的拆分，分布到不同的服务器上也成为了演进的必然。

- 实现原理

  大致可以分为客户端路由、代理层路由两种。

- 引入了哪些新的问题

  - 不同节点单表自增ID冲突
  - 跨库跨表的关联查询
  - 常见的AVG、MAX、MIN、COUNT、DISTINCT等函数不再适用
  - 引入了分布式事务
  - 数据如何分片、以及后续如何解决重新分片时做到数据平滑迁移

- 如何选择一个合适的分库分表框架

  在进行框架选择时考量的关键点有以下几点（不仅限于分库分表框架的选择）：

  - 接入的复杂度（上手难度）
  - 解决业务痛点的切合度（能否解决你的业务痛点）
  - 文档的完整性（良好的文档可以大大降低上手难度）
  - 社区的活跃度（出现了问题踩了坑能否及时解决）

在选择框架进行深入研究时也考量了很久，但就活跃度、文档完整性而言也只有以下两个相对比较靠谱的开源解决方案。

1. [incubator-shardingsphere](https://github.com/apache/incubator-shardingsphere)
2. [Mycat](https://github.com/MyCATApache/Mycat-Server)

## Sharding-JDBC原理解析系列

1. [总览](./overview.md)
2. [快速入门](./quickstart.md)
3. [框架是如何工作的](./how.md)
4. [常见的分片策略](./strategy.md)
5. [读写分离](./read_write_splitting.md)
7. [SQL解析及改写](./read_write_splitting.md)
9. [归并处理]()
10. [分布式事务](./distributed_trasaction.md)

参考文献：

[https://shardingsphere.apache.org/](https://shardingsphere.apache.org/)

[https://github.com/apache/incubator-shardingsphere](https://github.com/apache/incubator-shardingsphere)