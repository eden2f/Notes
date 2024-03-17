# MySQL 不可见索引VISIBLE和INVISIBLE

> [官方文档: Invisible Indexes](https://dev.mysql.com/doc/refman/8.0/en/invisible-indexes.html)
[AliSQL · 特性介绍 · 支持 Invisible Indexes](http://mysql.taobao.org/monthly/2017/07/03/)

注意，索引 VISIBLE 与 INVISIBLE 在 MySQL 8 之后才支持的
由于我本地的版本是 MySQL 8, 默认生成的建表语句中使用了默认的 VISIBLE index, 才发现了这一更新
简单来说，Invisible Indexes 的特点是：对优化器来说是不可见的，但是引擎内部还是会维护这个索引，并且不可见属性的修改操只改了元数据，所以可以非常快。 当我们发现某个索引不需要，想要去掉的话，可以先把索引设置为不可见，观察下业务的反应，如果一切正常，就可以 drop 掉；如果业务有受影响，那么说明这个索引删掉会有问题，就可以快速改回来。所以相对于 DROP/ADD 索引这种比较重的操作，Invisible Indexes 就会显得非常灵活方便。
Invisible Indexes 是 server 层的特性，和引擎无关，因此所有引擎（InnoDB, TokuDB, MyISAM）都可以使用。
## 以下是MySQL官网文档译文
MySQL支持不可见索引；也就是说，优化器未使用的索引。该功能适用于除主键（显式或隐式）以外的索引。
默认情况下，索引可见。为了控制可视性明确了新的索引，使用一个`VISIBLE`或 `INVISIBLE`关键字作为指标定义的一部分`[CREATE TABLE](https://dev.mysql.com/doc/refman/8.0/en/create-table.html)`， `[CREATE INDEX](https://dev.mysql.com/doc/refman/8.0/en/create-index.html)`或者 `[ALTER TABLE](https://dev.mysql.com/doc/refman/8.0/en/alter-table.html)`：
```
CREATE TABLE t1 (
  i INT,
  j INT,
  k INT,
  INDEX i_idx (i) INVISIBLE
) ENGINE = InnoDB;
CREATE INDEX j_idx ON t1 (j) INVISIBLE;
ALTER TABLE t1 ADD INDEX k_idx (k) INVISIBLE;
```
要更改现有索引的可见性，请 在 操作中使用 `VISIBLE`或`INVISIBLE`关键字`ALTER TABLE ... ALTER INDEX`：
```
ALTER TABLE t1 ALTER INDEX i_idx INVISIBLE;
ALTER TABLE t1 ALTER INDEX i_idx VISIBLE;
```
有关索引是可见还是不可见的信息可从 `[INFORMATION_SCHEMA.STATISTICS](https://dev.mysql.com/doc/refman/8.0/en/information-schema-statistics-table.html)`表或`[SHOW INDEX](https://dev.mysql.com/doc/refman/8.0/en/show-index.html)`输出中获得。例如：
```
mysql> SELECT INDEX_NAME, IS_VISIBLE
       FROM INFORMATION_SCHEMA.STATISTICS
       WHERE TABLE_SCHEMA = 'db1' AND TABLE_NAME = 't1';
+------------+------------+
| INDEX_NAME | IS_VISIBLE |
+------------+------------+
| i_idx      | YES        |
| j_idx      | NO         |
| k_idx      | NO         |
+------------+------------+
```
不可见的索引可以测试删除索引对查询性能的影响，而无需进行破坏性的更改，如果需要该索引，则必须撤消该更改。对于大型表，删除和重新添加索引可能会很昂贵，而使其不可见和可见则是快速的就地操作。
如果优化程序实际上需要或使用使索引变为不可见的索引，则有几种方法可以注意到缺少索引对表查询的影响：

- 对于包含引用不可见索引的索引提示的查询，会发生错误。
- 性能架构数据显示了受影响查询的工作量增加。
- 查询具有不同的 `[EXPLAIN](https://dev.mysql.com/doc/refman/8.0/en/explain.html)`执行计划。
- 查询出现在慢查询日志中，以前没有出现在查询日志中。

系统变量 的`[use_invisible_indexes](https://dev.mysql.com/doc/refman/8.0/en/switchable-optimizations.html#optflag_use-invisible-indexes)`标志`[optimizer_switch](https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_optimizer_switch)`控制优化器是否将不可见索引用于查询执行计划的构建。如果该标志是 `off`（缺省值），则优化器将忽略不可见索引（与引入此标志之前的行为相同）。如果该标志为 `on`，则不可见索引将保持不可见，但优化程序会在构建执行计划时将它们考虑在内。
使用`[SET_VAR](https://dev.mysql.com/doc/refman/8.0/en/optimizer-hints.html#optimizer-hints-set-var)`优化程序提示`[optimizer_switch](https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_optimizer_switch)`临时更新值 ，可以仅在单个查询期间启用不可见索引，如下所示：
```
mysql> EXPLAIN SELECT /*+ SET_VAR(optimizer_switch = 'use_invisible_indexes=on') */
     >     i, j FROM t1 WHERE j >= 50\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: t1
   partitions: NULL
         type: range
possible_keys: j_idx
          key: j_idx
      key_len: 5
          ref: NULL
         rows: 2
     filtered: 100.00
        Extra: Using index condition

mysql> EXPLAIN SELECT i, j FROM t1 WHERE j >= 50\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: t1
   partitions: NULL
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 5
     filtered: 33.33
        Extra: Using where
```
索引可见性不影响索引维护。例如，对于表行的更改，索引将继续更新，并且唯一索引可防止将重复项插入到列中，而不管索引是可见还是不可见。
没有显式主键的表如果`UNIQUE` 在`NOT NULL`列上有任何索引，则仍可能具有有效的隐式主键。在这种情况下，第一个这样的索引对表行施加与显式主键相同的约束，并且该索引不能不可见。考虑以下表定义：
```
CREATE TABLE t2 (
  i INT NOT NULL,
  j INT NOT NULL,
  UNIQUE j_idx (j)
) ENGINE = InnoDB;
```
该定义不包含显式主键，但`NOT NULL`列上的索引`j` 对行的约束与主键相同，并且不能使其不可见：
```
mysql> ALTER TABLE t2 ALTER INDEX j_idx INVISIBLE;
ERROR 3522 (HY000): A primary key index cannot be invisible.
```
现在，假设将一个显式主键添加到表中：
```
ALTER TABLE t2 ADD PRIMARY KEY (i);
```
显式主键不能不可见。此外，上的唯一索引`j`不再充当隐式主键，因此可以使其不可见：
```
mysql> ALTER TABLE t2 ALTER INDEX j_idx INVISIBLE;
Query OK, 0 rows affected (0.03 sec)
```
