# MySQL 类型隐式替换导致精度丢失

> [MySQL官方文档 : Type Conversion in Expression Evaluation](https://dev.mysql.com/doc/refman/5.7/en/type-conversion.html?spm=5176.100239.blogcont47339.5.1FTben)

## 前言
原来只知道, MySQL类型隐式替换会影响优化器对索引的选择, 由于遇到了一个隐式替换导致的精度丢失的 Bug ,引起了我对隐式替换的实现逻辑的好奇心.
为什么隐式替换会影响精度吗? 不是类似 String.valueOf() 处理吗?
## 类型隐式替换导致精度丢失例子
执行SQL 1 :
```
select * from type_conversion_test where business_order_id = '210517130303013756';
```
查询结果 1 :

| id | business_order_id |
| --- | --- |
| 3 | 210517130303013756 |

执行SQL 2 :
```
select * from type_conversion_test where business_order_id = 210517130303013756;
```
查询结果 2 :

| id | business_order_id |
| --- | --- |
| 1 | 210517130303013752 |
| 2 | 210517130303013770 |
| 3 | 210517130303013756 |
| 4 | 210517130303013767 |
| 5 | 210517130303013773 |

由上面的两个SQL看到, 由于where条件的业务订单id筛选项没有添加引号, 导致了精度丢失问题. `210517130303013756`在表中是唯一的, 但是查出来多条数据.
如需构建演示环境请执行如下SQL
```
# 初始化演示表结构
CREATE TABLE `type_conversion_test` (
   `id` int NOT NULL AUTO_INCREMENT,
   `business_order_id` varchar(45) DEFAULT NULL,
   PRIMARY KEY (`id`),
   KEY `idx_business_order_id` (`business_order_id`)
 ) ENGINE=InnoDB AUTO_INCREMENT=25 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
# 初始化演示数据
INSERT INTO `db_email_util`.`type_conversion_test`
(`business_order_id`)
VALUES
("210517130303013752"),("210517130303013770"),("210517130303013756"),("210517130303013767"),("210517130303013773");
```
## 隐式数字到字符串转换的字符集
这里贴出了数字类型相关的替换规则, 想了解更多请查阅官方文档
有关隐式数字到字符串转换的字符集以及适用于`CREATE TABLE ... SELECT` 语句的修改规则，请参阅本节后面的信息。
以下规则描述了比较操作如何发生转换：

-  如果一个或两个参数均为`NULL`，则比较的结果为`NULL`，除了`NULL`-safe `[<=>](https://dev.mysql.com/doc/refman/5.7/en/comparison-operators.html#operator_equal-to)` 相等比较运算符。对于`NULL <=> NULL`，结果为true。无需转换。 
-  如果比较操作中的两个参数都是字符串，则将它们作为字符串进行比较。 
-  如果两个参数都是整数，则将它们作为整数进行比较。 
-  如果不将十六进制值与数字进行比较，则将其视为二进制字符串。 
-  如果参数之一是a `[TIMESTAMP](https://dev.mysql.com/doc/refman/5.7/en/datetime.html)`或 `[DATETIME](https://dev.mysql.com/doc/refman/5.7/en/datetime.html)`column，而另一个参数是常量，则在执行比较之前，该常量将转换为时间戳。这样做是为了使ODBC更友好。对于的参数，此操作未完成 `[IN()](https://dev.mysql.com/doc/refman/5.7/en/comparison-operators.html#operator_in)`。为了安全起见，在进行比较时，请始终使用完整的日期时间，日期或时间字符串。例如，要在`[BETWEEN](https://dev.mysql.com/doc/refman/5.7/en/comparison-operators.html#operator_between)`与日期或时间值一起使用时获得最佳结果 ，请使用`[CAST()](https://dev.mysql.com/doc/refman/5.7/en/cast-functions.html#function_cast)`将值显式转换为所需的数据类型。
一个或多个表中的单行子查询不视为常量。例如，如果子查询返回要与`[DATETIME](https://dev.mysql.com/doc/refman/5.7/en/datetime.html)` 值进行比较的整数，则比较将作为两个整数完成。整数不转换为时间值。要将操作数作为`[DATETIME](https://dev.mysql.com/doc/refman/5.7/en/datetime.html)`值进行比较 ，可 `[CAST()](https://dev.mysql.com/doc/refman/5.7/en/cast-functions.html#function_cast)`用于将子查询值显式转换为`[DATETIME](https://dev.mysql.com/doc/refman/5.7/en/datetime.html)`。 
-  如果参数之一是十进制值，则比较取决于另一个参数。如果另一个参数是十进制或整数值，则将参数作为十进制值进行比较；如果另一个参数是浮点值，则将参数作为浮点值进行比较。 
-  在所有其他情况下，将参数作为浮点数（实数）进行比较。例如，将字符串和数字操作数进行比较，将其作为浮点数的比较。 

有关将值从一种时间类型转换为另一种时间类型的信息，请参见[第11.2.8节“日期和时间类型之间的转换”](https://dev.mysql.com/doc/refman/5.7/en/date-and-time-type-conversion.html)。
## 结论
```
select * from type_conversion_test where business_order_id = 210517130303013756;
```
由于 `210517130303013756` 没有加引号, 默认将其作为浮点数处理, 所以在 210517130303013756 转浮点数的时候, 导致了精度丢失问题。
