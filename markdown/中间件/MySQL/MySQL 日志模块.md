# MySQL 事务隔离

> 原文《MySQL实战45讲》

### redo log （重做日志）
redo log 是 InnoDB 引擎特有的日志，主要是为了实现 crash-safe能力。保证即使数据库发生异常重启，之前提交的记录都不会丢失，这个能力称为crash-safe。
当有一条记录需要更新的时候，InnoDB 引擎就会先把记录写到 redo log 里面，并更新内存，这个时候更新就算完成了。同时，InnoDB 引擎会在适当的时候，将这个操作记录更新到磁盘里面，而这个更新往往是在系统比较空闲的时候做。InnoDB 的 redo log 是固定大小的，从头开始写，写到末尾就又回到开头循环写。
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240303122537.png#id=xNKsQ&originHeight=856&originWidth=1142&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
write pos 是当前记录的位置，一边写一边后移，写到文件末尾就重头开始写。checkpoint 是当前要擦除的位置，也是往后推移并且循循环的，擦除记录前要把记录更新到数据文件。write pos 和 checkpoint 之间的空间可以存入新的更新，如果 write pos 追上了 checkpoint ，这个时候不能再执行新的更新了，需要把 checkpoint 推进一下。
### binlog （归档日志）
redo log 是 InnoDB 引擎特有的日志，而Server层也有自己的日志，为binlog（归档日志）。binlog日志只能用于归档，没有crash-safe的能力。binlog 是 MySQL 的 Server 层实现的，所有引擎都可以使用。
### redo log 和 binlog 的区别

-  redo log 是物理日志，记录的是“在某个数据页上做了什么修改”；binlog 是逻辑日志，记录的是这个语句的原始逻辑，比如“给某一行的 某字段数值+1”。 
-  redo log 是循环写的，空间固定的，会用完；binlog 是追加写入的（binlog 文件写道一定大小后会切换到下一个，不会覆盖以前的日志）。 
-  一个更新语句的的执行流程图，图中浅色框表示是在 InnoDB 内部执行的，深色框表示是在执行器中执行的。 
   -  更新语句 
```
update T set c=c+1 where ID=2;
```

   -  执行流程图![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240303122624.png#id=rDuDR&originHeight=1522&originWidth=1142&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
### MySQL 相关设置项

-  innodb_flush_log_at_trx_commit 这个参数设置成1的时候，表示每次事务的 redo log 都直接 flush（刷到磁盘）。这个参数设置成1，可以保证 MySQL 异常重启之后数据不会丢失。 
   - 如果innodb_flush_log_at_trx_commit设置为0，将事务的变更每秒一次地写入 redo log 中，并且 redo log 的flush(刷到磁盘)操作同时进行。该模式下，在事务提交的时候，不会主动触发写入磁盘的操作。
   - 如果innodb_flush_log_at_trx_commit设置为2，每次事务提交时MySQL都会把事务的数据写入 redo log。但是flush(刷到磁盘)操作并不会同时进行。该模式下,MySQL会每秒执行一次 flush(刷到磁盘)操作。
   - 注意：由于进程调度策略问题,这个“每秒执行一次 flush(刷到磁盘)操作，并不是保证100%的“每秒”。
-  sync_binlog 这个参数设置成1的时候，表示每次失误的 binlog 都持久化到磁盘。这个参数设置成1，可以保证MySQL 异常重启之后 binlog 不丢失。 
   - sync_binlog 默认为0，表示 MySQL 不控制 binlog 的刷新，由文件系统自己控制它的缓存刷新。这时候的性能是最好的，但是风险也是最大的。因为一旦发生系统 crash，在 binlog_cache 中的所有 binlog 信息都会丢失。
   - 如果 sync_binlog > 0, 设为 n，表示每 n 次事务提交， MySQL 调用文件系统的刷新操作将 binlog 日志持久化。当 sync_binlog = 1 的时候是最安全的，表示每次事务提交， MySQL 都会把 binlog 持久化，但是这时候的性能损耗也是最大的。这样的话，在数据库所在的主机操作系统损坏或者突然掉电的情况下，系统才有可能丢失一个事务数据。有些时候，可以将 sync_binlog 设置为 0 或者 100，牺牲一定的一致性，以获得更高的并发和性能。
   - 注意：如果启用了 autocommit，那么每一个语句statement就会有一次写操作；否则每个事务对应一个写操作。
### binlog 使用

1.  开启 binlog 日志(Docker安装的MySQL) 
```shell
# 进入mysql容器
docker exec -it [mysql容器id] /bin/bash
# 进入配置文件目录下
cd /etc/mysql/mysql.conf.d/
# 开启binlog
echo 'log-bin=/var/lib/mysql/mysql-bin' >> mysqld.cnf
echo 'server-id=123454' >> mysqld.cnf
# 修改binlog格式
echo 'binlog-format=ROW' >> mysqld.cnf
```
