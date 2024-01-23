# MySQL UPDATE更新语法问题

## 问题

提了SQL工单，执行后成功后，发现数据没有变更。

``` sql
UPDATE t_user 
SET status = '20' AND address_id = 'YYY' 
WHERE user_id IN ('111', '222', '333')
```

## 原因

UPDATE 语法问题，将 "," 错写成 AND，执行不会报错，只是不会修改数据。

正确的UPDATE语法如下 :

``` sql
UPDATE table_name
SET column1 = value1, column2 = value2, ...
WHERE condition;
```

## 疑问

SQL明明不符合SQL格式，为什么数据库没有报异常？答案：MySQL认为这是一句正常的SQL！

MySQL是这么理解这个SQL的 :

``` sql
UPDATE t_user 
SET status = ('20' AND address_id = 'YYY')
WHERE user_id IN ('111', '222', '333')
```

``` sql
-- 执行
select status = ('20' AND address_id = 'YYY'); 
-- 输出
0
```

所以不仅没有执行异常，还更新成功的将 status 更新为 0。惊不惊喜～意不意外～
