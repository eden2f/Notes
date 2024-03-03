# MySQL 如何判断一个数据库是否出问题了

> 原文《MySQL实战45讲》

## 前言
我第一次使用 select 1 ，是在项目里面引入 Druid 时看到了连接的有效性判断配置，部分配置如下：
```xml
······
<property name="validationQuery" value="select 1" />
<property name="testWhileIdle" value="true" />
<property name="testOnBorrow" value="false" />
<property name="testOnReturn" value="false" />
······
```
在 Druid 的参考配置中，我们可以看到，通过 “select 1”，连接池就可以判断连接是否有效了。但是 "select 1 " 返回了，就表示主库没问题吗？ 经过学习，发现还真不是这么简单。
## select 1 判断
实际上，select 1 成功返回，只能说明这个库的进程还存在，并不能说明数据库没有问题，以一下场景为例。
```
# 设置 innodb 的并发线程数
set global innodb_thread_concurrency=3;
# 建表语句
CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `c` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB;
# 数据初始化
insert into t values(1,1)
```
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240303125405.png#id=IZDn3&originHeight=290&originWidth=932&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
设置 innodb_thread_concurrency  是为了控制 InnoDB 的并发线程上限。也就是说，一旦并发线程数达到这个值，InnoDB就不会马上执行新的请求，而是等待线程资源，直到当前正在执行的线程退出。InnoDB中，innodb_thread_concurrency  默认为0，表示不限制并发线程数。但是不设置并发线程数肯定是不行的，因为机器的CPU处理能力有限，大量线程切换处理的过程中，上下文切换的成本太高。
在 session D 中，select 1；是可以 执行成功的，但实际查询语句却被 blocked 。也就是说，这个场景下，使用 select 1；并不能检测到问题。
#### 不要混淆并发连接和并发查询
show processlist 的结果里，看到的是并发连接，并发连接多只是多占用一些内存，实际消耗CPU资源的是并发查询。
而“当前正在执行”的语句，才是我们所说的并发查询。
#### 在线程进入锁等待后，并发线程会减一
热点数据更新和死锁检测的时候，如果 innodb_thread_concurrency  设置的太小，同时该热点行更新过程中，发生了死锁等问题，那等待更新该行的线程岂不是很快就达到 innodb_thread_concurrency  了，导致数据库没有空闲的资源可以执行其他请求？
实际上，等待行锁的（也包括间隙锁）的线程是不算会算在 innodb_thread_concurrency  里面的。MySQL 为什么要这么设计呢？ 因为，进入锁等待的线程已经不再消耗CPU了，更重要的是，这么设计可以避免整个数据系统被锁死。
举个例子：innodb_thread_concurrency   设置为 128 ，此时有128个线程等待同一个行锁；但是这个时候还是可以继续处理新的请求的，因为等待行锁的线程并不会占用 innodb_thread_concurrency  。当然，但等待中的线程真正的执行查询，就会占用 innodb_thread_concurrency 了。
## 查表判断
为了能够检测InnoDB并发线程数过多而导致系统不可用的情况，我们需要找一个访问InnoDB的场景。常见的做法是，在系统库（mysql库）里创建一个表，比如命名为 health_check, 里面放一行数据，然后定期执行。
```
mysql> select * from mysql.health_check;
```
使用这个方法，我们可以检测出由于并发线程过多导致的数据库不可用的情况。但是，我们马上还会碰到下一个问题，即：磁盘空间满了以后，这个方法就无能为力了。
对于更新操作的事务，提交后是需要写binlog的，如果binlog所在的磁盘占用达到了100%了，那么所有的更新语句提交的commit语句都会被堵住。但是，此时系统还是可以正常执行查询请求的。
所以，我们需要把查询语句改成更新语句后，才能检测到该问题。
## 更新判断
既然要更新，就要放个有意义的字段，常见做法是放一个 timestamp 字段，用来表示最后一次执行检测的时间。这条更新语句类似于：
```
mysql> update mysql.health_check set t_modified=now();
```
节点可用性的检测都应该包含主库和备库。如果用更新来检测主库的话，那么备库也要进行更新检测。
但，备库的检测也是要写 binlog 的。由于我们一般会把数据库 A 和 B 的主备关系设计为双 M 结构，所以在备库 B 上执行的检测命令，也要发回给主库 A。
但是，如果主库 A 和备库 B 都用相同的更新命令，就可能出现行冲突，也就是可能会导致主备同步停止。所以，现在看来 mysql.health_check 这个表就不能只有一行数据了。
为了让主备之间的更新不产生冲突，我们可以在 mysql.health_check 表上存入多行数据，并用 A、B 的 server_id 做主键。
```
mysql> CREATE TABLE `health_check` (
  `id` int(11) NOT NULL,
  `t_modified` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB;

/* 检测命令 */
insert into mysql.health_check(id, t_modified) values (@@server_id, now()) on duplicate key update t_modified=now();
```
由于 MySQL 规定了主库和备库的 server_id 必须不同（否则创建主备关系的时候就会报错），这样就可以保证主、备库各自的检测命令不会发生冲突。
更新判断是一个相对比较常用的方案了，不过依然存在一些问题。其中，“判定慢”一直是让 DBA 头疼的问题。你一定会疑惑，更新语句，如果失败或者超时，就可以发起主备切换了，为什么还会有判定慢的问题呢？
其实，这里涉及到的是服务器 IO 资源分配的问题。
你可以设想一个日志盘的 IO 利用率已经是 100% 的场景。这时候，整个系统响应非常慢，已经需要做主备切换了。
但是你要知道，IO 利用率 100% 表示系统的 IO 是在工作的，每个请求都有机会获得 IO 资源，执行自己的任务。而我们的检测使用的 update 命令，需要的资源很少，所以可能在拿到 IO 资源的时候就可以提交成功，并且在超时时间 N 秒未到达之前就返回给了检测系统。
## 内部统计
我们上面说的所有方法，都是基于外部检测的。外部检测天然有一个问题，就是随机性。因为，外部检测都需要定时轮询，所以系统可能已经出问题了，但是却需要等到下一个检测发起执行语句的时候，我们才有可能发现问题。而且，如果你的运气不够好的话，可能第一次轮询还不能发现，这就会导致切换慢的问题。
针对磁盘利用率这个问题，如果 MySQL 可以告诉我们，内部每一次 IO 请求的时间，那我们判断数据库是否出问题的方法就可靠得多了。MySQL 5.6 版本以后提供的 performance_schema 库，就在 file_summary_by_event_name 表里统计了每次 IO 请求的时间。
file_summary_by_event_name 表里有很多行数据，我们先来看看event_name='wait/io/file/innodb/innodb_log_file’这一行。
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240303125505.png#id=bAuAC&originHeight=518&originWidth=1240&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
图中这一行表示统计的是 redo log 的写入时间，第一列 EVENT_NAME 表示统计的类型。
接下来的三组数据，显示的是 redo log 操作的时间统计。
第一组五列，是所有 IO 类型的统计。其中，COUNT_STAR 是所有 IO 的总次数，接下来四列是具体的统计项， 单位是皮秒；前缀 SUM、MIN、AVG、MAX，顾名思义指的就是总和、最小值、平均值和最大值。
第二组六列，是读操作的统计。最后一列 SUM_NUMBER_OF_BYTES_READ 统计的是，总共从 redo log 里读了多少个字节。
第三组六列，统计的是写操作。
最后的第四组数据，是对其他类型数据的统计。在 redo log 里，你可以认为它们就是对 fsync 的统计。
除了对 'wait/io/file/innodb/innodb_log_file’ 的统计，还有其他总共有46行统计信息，需要用到的时候再详细了解吧；
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240303125529.png#id=OYNYJ&originHeight=304&originWidth=583&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
因为我们每一次操作数据库，performance_schema 都需要额外地统计这些信息，所以我们打开这个统计功能是有性能损耗的。如果打开所有的 performance_schema 项，性能大概会下降 10% 左右。所以，我建议你只打开自己需要的项进行统计。你可以通过下面的方法打开或者关闭某个具体项的统计。
如果要打开 redo log 的时间监控，你可以执行这个语句：
```
mysql> update setup_instruments set ENABLED='YES', Timed='YES' where name like '%wait/io/file/innodb/innodb_log_file%';
```
假设，现在你已经开启了 redo log 和 binlog 这两个统计信息，那要怎么把这个信息用在实例状态诊断上呢？很简单，你可以通过 MAX_TIMER 的值来判断数据库是否出问题了。比如，你可以设定阈值，单次 IO 请求时间超过 200 毫秒属于异常，然后使用类似下面这条语句作为检测逻辑。
```
mysql> select event_name,MAX_TIMER_WAIT  FROM performance_schema.file_summary_by_event_name where event_name in ('wait/io/file/innodb/innodb_log_file','wait/io/file/sql/binlog') and MAX_TIMER_WAIT>200*1000000000;
```
发现异常后，取到你需要的信息，再通过下面这条语句把之前的统计信息清空。这样如果后面的监控中，再次出现这个异常，就可以加入监控累积值了。
```
mysql> truncate table performance_schema.file_summary_by_event_name;
```
