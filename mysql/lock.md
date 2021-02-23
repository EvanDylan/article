# MySQL锁和MVCC

## 锁的一些概念

### 共享锁和排它锁（行级别）

共享锁（Shared Lock）

排它锁 （exclusive lock）

### 意向锁（表级别）

意向共享锁 (intention shared lock)

意向排它锁 （intention exclusive lock）

### 自增锁（AUTO-INC Lock）

### 记录锁 （Record Lock）

单个行记录上的锁

### 间隙锁 （Gap Lock）

锁定一个范围，但不包含记录本身

### 临键锁  （Next-Key lock）

1. 锁定一个范围，并且锁定记录本身（临建锁 = 记录锁 + 临建锁）
2. MySQL InnoDB对于行的查询都是采用这种锁定算法，但是当查询的索引包含有唯一的属性时，该锁会进行优化降级为Record Lock
3. 设计的目的是为了解决幻读的问题

## 多版本控制

MVCC + undo log

read commit 始终读最新的undo log

Repeated Read 始终读当前事务id对应的快照版本