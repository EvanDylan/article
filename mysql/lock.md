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

### 间隙锁 （Gap Lock）

### 临键锁  （Next-Key lock）

临建锁 = 记录锁 + 临建锁

## 多版本控制

MVCC + undo log

read commit 始终读最新的undo log

Repeated Read 始终读当前事务id对应的快照版本