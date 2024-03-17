# ElasticSearch存储设计与MySQL数据同步方案

## ElasticSearch 存储设计
ElasticSearch中index约等于MySQL的database，type约等于MySQL的table；
虽然ElasticSearch支持一个index多个type，但是官方并不推荐如此使用，而且多个type底层其实也是用一张大表进行存储的。
所以，ElasticSearch 中存储的数据用一个type来存储就可以了。使用nested类型保存一对多关系。
## MySQL数据同步
### 1. MySQL & ElasticSearch双写

- 实现思路 ：在业务 insert ＆ update MySQL的地方同时  insert ＆ update ElasticSearch
- 优点 ：实现简单
- 缺点 ： 业务代码侵入；能力无法复用；业务强耦合；影响业务处理性能；
- 优化做法：配合MQ提高吞吐量、改同步为异步
- 推荐指数 ： 不可取
### 2. 定时 MySQL SELECT

-  实现思路 ：数据库的相关表中增加是否需要同步的标识字段，例如现有的类型为 timestamp 的 update_time 字段，任何crud操作都会导致该字段的时间发生变化；增加一个定时器程序，让该程序按一定的时间周期扫描指定的表，把该时间段内发生变化的数据提取到 ElasticSearch ； 
-  优点 ：实现简单；没有侵入；没有硬编码； 
-  缺点 ： 时效性较差；对 MySQL Delete 操作无能为力；增大 MySQL 的压力； 
-  优化做法：给标志字段添加索引；从从库进行读取；使用生产者消费者模式提高同步性能； 
-  推荐指数 ： 一般 
### 3. Binlog 解析

-  实现思路 ：读取 MySQL 的 Binlog 日志，获取指定表的日志信息；根据信息同步到 ES； 
-  优点 ：没有侵入；没有硬编码；性能高；业务解耦； 
-  缺点 ： 实现相对复杂； 
-  优化做法：配合MQ，使用生产者消费者模式提高同步性能；采用 MySQL 主-从-从模式； 
-  推荐指数 : 推荐 
   - 开源组件：阿里 canal
-  实现依赖 
   - MySQL , 需要先开启 Binlog 写入功能，配置 binlog-format 为 ROW 模式
   - 授权 MySQL 连接账号，使其具有作为 MySQL slave 的权限
