# 配置合集

## 缓存相关

- `innodb_buffer_pool_size`：数据及索引的内存缓冲区域（默认为128M）。调优的情况下可以设置为系统剩余可用内存的80%都分配给该区域。该区域使用LRU的算法，将最近常使用到热点数据放到链表靠前的位置。该区域内存越大，查询时命中的缓存的概率也就越大，查询的性能也就越高。
- `sort_buffer_size` ：当连接被创建的时候内存就已经被分配（默认大小为256K）。在使用到额外的**filesort**时，需要对结果集在内存里进行二次排序，如果可以直接使用索引则优先使用索引进行排序，毕竟索引已经是有序的不需要进行二次排序。
- `join_buffer_size`：当发生关联查询的时候在某些场景下会使用到该内存（默认大小256K），以减少因为关联查询产生的嵌套循环次数。详情请参考官网[链接](https://dev.mysql.com/doc/refman/5.7/en/nested-loop-joins.html)。
- `binlog_cache_size`：binlog的缓存大小（32KB）。在binlog没有被落地到磁盘前需要内存区域做缓冲，为了安全性基本上都是一次事务提交时直接将binlog落盘，所以太大的内存没有意义。

## 主从半同步复制

1. master配置
   `log-bin`：指定binlog的日志名。
   `server-id`：指定可以在集群中唯一标识当前机器的数字（1、2、3这种数字即可）。

   `plugin-load=rpl_semi_sync_master=semisync_master.so`，固定配置表示主库启用半同步复制插件。

   `rpl_semi_sync_master_enabled=1`，主库开启半同步复制。

   `rpl_semi_sync_master_timeout`，主库等待从库同步完成等待时间（默认10s），超时从库还未完成同步则进入异步同步工作状态，主从此时的延迟会比较高了。

   `rpl_semi_sync_master_wait_point`。有`AFTER_SYNC`、`AFTER_COMMIT`可选，默认为`AFTER_SYNC`。

   - `AFTER_SYNC`等待从库完成同步才是为完成
   - `AFTER_COMMIT`等待主库完成commit时即视为完成，此时从节点还没完成同步操作。如果主库宕机，从库切为主库会导致部分数据丢失。

2. slave配置

   `relay-log`：指定中继日志名称。从节点接收到主节点的binlog时写入本地的文件。

   `relay-log-index`：中继日志索引

   `server-id`：指定可以在集群中唯一标识当前机器的数字（1、2、3这种数字即可），注意不要和主服务器重复。

   `plugin-load=rpl_semi_sync_slave=semisync_slave.so`，固定配置表示从库启用半同步复制插件。

   `rpl_semi_sync_slave_enabled=1`，从库开启半同步复制。

## 慢日志

- `slow_query_log=1`启用慢日志记录功能
- `slow_query_log_file=/var/log/mysql/slow_queries.log`指定慢日志记录位置
- `long_query_time=1`当查询超过该值时则记录下来，单位ms。

## 查询缓存

- `query_cache_typ=0`（默认值为0，表示关闭状态），如果需要开启值为1即可。
- `query_cache_size=1M`（默认大小为1M）。

## 其他

- `innodb_write_io_threads`、`innodb_read_io_threads `后台用于处理数据读写的线程数量，两者默认为4。当CPU核心数比较多时，可以适当调整。
- `innodb_purge_threads`用于清理undolog日志工作的线程（事务已经提交过的才可以清理），默认大小为1.
- `innodb_flush_log_at_trx_commit`，控制MySQL刷盘策略，默认值为1。
  - 当取值为0时，每秒一次地频率将日志写入缓存，刷盘的动作由os cache控制。
  - 当取值为1时，每次事务提交都会主动将日志缓存落盘。
  - 当取值为2时，每秒一次地频率主动将日志缓存落盘。

- `sync_binlog`，控制MySQL对binlog刷盘策略，版本>= 5.7.7时默认值为1。
  - 当取值为0时，有操作系统控制刷盘动作。
  - 当取值为1时，每次binlog写入动作都会主动将日志缓存落盘。
  - 当取值>1时，进行n次binlog写入动作才会主动将日志缓存落盘。
- `log-error=/var/log/mysql/error.log`，开启错误日志。
- `max_connections`，最大可建立客户端连接数量（默认为151）。
- `max_allowed_packet`，最大传输的包大小（默认为4M）。