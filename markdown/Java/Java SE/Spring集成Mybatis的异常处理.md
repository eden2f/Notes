# Spring集成Mybatis的异常处理

> [博客原文 : Spring3 Mybatis 异常处理](https://my.oschina.net/yyjava/blog/160380)

## 前言
由于线上报了一个主键冲突异常, 由于该异常时业务无损的, 所以想将该异常catch, 发现该异常是无法直接捕获的, 需要用 DuplicateKeyException 捕获.
```java
// 原始异常
com.mysql.jdbc.exceptions.jdbc4.MySQLIntegrityConstraintViolationException
// 可以捕获的异常
org.springframework.dao.DuplicateKeyException
```
## Spring Mybatis 异常处理
通常在 Dao 层将所有异常都转嫁到 Spring 的 RuntimeException 体系中来　－DataAccessException
Spring的DAO框架没有抛出与特定技术相关的异常，例如SQLException或HibernateException，抛出的异常都是 与特定技术无关的org.springframework.dao.DataAccessException类的子类，避免系统与某种特殊的持久层实现耦 合在一起。DataAccessException是RuntimeException，是一个无须检测的异常，不要求代码去处理这类异常，遵循了 Spring的一般理念：异常检测会使代码到处是不相关的catch或throws语句，使代码杂乱无章；并且 NestedRuntimeException的子类，是可以通过NestedRuntimeException的getCause（）方法获得导致该异 常的另一个异常。
Spring的DAO异常层次

| 异常 | 何时抛出 |
| --- | --- |
| CleanupFailureDataAccessException | 一项操作成功地执行，但在释放数据库资源时发生异常（例如，关闭一个Connection） |
| DataAccessResourceFailureException | 数据访问资源彻底失败，例如不能连接数据库 |
| DataIntegrityViolationException | Insert或Update数据时违反了完整性，例如违反了惟一性限制 |
| DataRetrievalFailureException | 某些数据不能被检测到，例如不能通过关键字找到一条记录 |
| DeadlockLoserDataAccessException | 当前的操作因为死锁而失败 |
| IncorrectUpdateSemanticsDataAccessException | Update时发生某些没有预料到的情况，例如更改超过预期的记录数。当这个异常被抛出时，执行着的事务不会被回滚 |
| InvalidDataAccessApiusageException | 一个数据访问的JAVA API没有正确使用，例如必须在执行前编译好的查询编译失败了 |
| invalidDataAccessResourceUsageException | 错误使用数据访问资源，例如用错误的SQL语法访问关系型数据库 |
| OptimisticLockingFailureException | 乐观锁的失败。这将由ORM工具或用户的DAO实现抛出 |
| TypemismatchDataAccessException | Java类型和数据类型不匹配，例如试图把String类型插入到数据库的数值型字段中 |
| UncategorizedDataAccessException | 有错误发生，但无法归类到某一更为具体的异常中 |

Spring的DAO异常层次是如此的细致缜密，服务对象能够精确地选择需要捕获哪些异常，捕获的异常对用户更有用的信息，哪些异常可以让她继续在调用堆栈中向上传递。
于是，我们在DAO中只需要抛出这个运行时异常，我们就可以在调用DAO方法时捕获这个异常即可. 如下 :
```java
try {
    // DAO接口调用
} catch(DataAccessException e) {
    log.error("未知异常", e)
}
```
