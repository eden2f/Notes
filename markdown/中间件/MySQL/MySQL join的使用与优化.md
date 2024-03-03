# MySQL join的使用与优化

> 原文《MySQL实战45讲》

## 前言
在实际生产中，关于 join 语句使用的问题，一般会集中在以下两类：

1.  我们 DBA 不让使用 join，使用 join 有什么问题呢？ 
2.  如果有两个大小不同的表做 join，应该用哪个表做驱动表呢？
为了便于量化分析，我还是创建两个表 t1 和 t2 来和你说明。可以看到，这两个表都有一个主键索引 id 和一个索引 a，字段 b 上无索引。存储过程 idata() 往表 t2 里插入了 1000 行数据，在表 t1 里插入的是 100 行数据。 
```
CREATE TABLE `t2` (
  `id` int(11) NOT NULL,
  `a` int(11) DEFAULT NULL,
  `b` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `a` (`a`)
) ENGINE=InnoDB;

drop procedure idata;
delimiter ;;
create procedure idata()
begin
  declare i int;
  set i=1;
  while(i<=1000)do
    insert into t2 values(i, i, i);
    set i=i+1;
  end while;
end;;
delimiter ;
call idata();

create table t1 like t2;
insert into t1 (select * from t2 where id<=100)
```
## Index Nested-Loop Join
我们来看一下这个语句：
```
select * from t1 straight_join t2 on (t1.a=t2.a);
```
> 如果直接使用 join 语句，MySQL 优化器可能会选择表 t1 或 t2 作为驱动表，这样会影响我们分析 SQL 语句的执行过程。所以，为了便于分析执行过程中的性能问题，我改用 straight_join 让 MySQL 使用固定的连接方式执行查询，这样优化器只会按照我们指定的方式去 join。在这个语句里，t1 是驱动表，t2 是被驱动表。

现在，我们来看一下这条语句的 explain 结果。
![](https://upload-images.jianshu.io/upload_images/4139030-80bc7c5f122c928c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240#id=BsdY3&originHeight=163&originWidth=1394&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)

1.  从表 t1 中读入一行数据 R； 
2.  从数据行 R 中，取出 a 字段到表 t2 里去查找； 
3.  取出表 t2 中满足条件的行，跟 R 组成一行，作为结果集的一部分； 
4.  重复执行步骤 1 到 3，直到表 t1 的末尾循环结束。
这个过程是先遍历表 t1，然后根据从表 t1 中取出的每行数据中的 a 值，去表 t2 中查找满足条件的记录。在形式上，这个过程就跟我们写程序时的嵌套查询类似，并且可以用上被驱动表的索引，所以我们称之为“Index Nested-Loop Join”，简称 NLJ。
它对应的流程图如下所示： 

![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240303130648.png#id=cP9CR&originHeight=880&originWidth=1142&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
在这个流程里：

1. 对驱动表 t1 做了全表扫描，这个过程需要扫描 100 行；
2. 而对于每一行 R，根据 a 字段去表 t2 查找，走的是树搜索过程。由于我们构造的数据都是一一对应的，因此每次的搜索过程都只扫描一行，也是总共扫描 100 行；
3. 所以，整个执行流程，总扫描行数是 200。
### 能不能使用 join
假设不使用 join，那我们就只能用单表查询。我们看看上面这条语句的需求，用单表查询怎么实现。

1. 执行select * from t1，查出表 t1 的所有数据，这里有 100 行；
2.  循环遍历这 100 行数据： 
   1. 从每一行 R 取出字段 a 的值 $R.a；
   2. 执行select * from t2 where a=$R.a；
   3. 把返回的结果和 R 构成结果集的一行。

可以看到，在这个查询过程，也是扫描了 200 行，但是总共执行了 101 条语句，比直接 join 多了 100 次交互。除此之外，客户端还要自己拼接 SQL 语句和结果。显然，这么做还不如直接 join 好。 
### 怎么选择驱动表
在这个 join 语句执行过程中，驱动表是走全表扫描，而被驱动表是走树搜索。
假设被驱动表的行数是 M。每次在被驱动表查一行数据，要先搜索索引 a，再搜索主键索引。每次搜索一棵树近似复杂度是以 2 为底的 M 的对数，记为 log2M，所以在被驱动表上查一行的时间复杂度是 2*log2M。
假设驱动表的行数是 N，执行过程就要扫描驱动表 N 行，然后对于每一行，到被驱动表上匹配一次。
因此整个执行过程，近似复杂度是 N + N*2*log2M。可以这么理解：N 扩大 1000 倍的话，扫描行数就会扩大 1000 倍；而 M 扩大 1000 倍，扫描行数扩大不到 10 倍。
显然，N 对扫描行数的影响更大，因此应该让小表来做驱动表。
## Simple Nested-Loop Join
上面的例子走了被驱动表的索引，接下来，我们再看看被驱动表用不上索引的情况。现在，我们把 SQL 语句改成这样：
```
select * from t1 straight_join t2 on (t1.a=t2.b);
```
由于表 t2 的字段 b 上没有索引，因此再用图 2 的执行流程时，每次到 t2 去匹配的时候，就要做一次全表扫描,这个算法叫做“Simple Nested-Loop Join”。
但是，这样算来，这个 SQL 请求就要扫描表 t2 多达 100 次，总共扫描 100*1000=10 万行。当然，MySQL 也没有使用这个 Simple Nested-Loop Join 算法，而是使用了另一个叫作“Block Nested-Loop Join”的算法，简称 BNL。
## Block Nested-Loop Join
当被驱动表上没有可用的索引，算法的流程是这样的：

1.  把表 t1 的数据读入线程内存 join_buffer 中，由于我们这个语句中写的是 select *，因此是把整个表 t1 放入了内存； 
2.  扫描表 t2，把表 t2 中的每一行取出来，跟 join_buffer 中的数据做对比，满足 join 条件的，作为结果集的一部分返回。
这个过程的流程图如下： 

![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240303130753.png#id=iE6wT&originHeight=880&originWidth=1142&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
对应地，这条 SQL 语句的 explain 结果如下所示：
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240303130818.png#id=bR8Bp&originHeight=140&originWidth=1500&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
可以看到，在这个过程中，对表 t1 和 t2 都做了一次全表扫描，因此总的扫描行数是 1100。由于 join_buffer 是以无序数组的方式组织的，因此对表 t2 中的每一行，都要做 100 次判断，总共需要在内存中做的判断次数是：100*1000=10 万次。
前面我们说过，如果使用 Simple Nested-Loop Join 算法进行查询，扫描行数也是 10 万行。因此，从时间复杂度上来说，这两个算法是一样的。但是，Block Nested-Loop Join 算法的这 10 万次判断是内存操作，速度上会快很多，性能也更好。
接下来，我们来看一下，在这种情况下，应该选择哪个表做驱动表。
假设小表的行数是 N，大表的行数是 M，那么在这个算法里：

1.  两个表都做一次全表扫描，所以总的扫描行数是 M+N； 
2.  内存中的判断次数是 M*N。
可以看到，调换这两个算式中的 M 和 N 没差别，因此这时候选择大表还是小表做驱动表，执行耗时是一样的。 要是表 t1 是一个大表，join_buffer 放不下怎么办呢？join_buffer 的大小是由参数 join_buffer_size 设定的，默认值是 256k。如果放不下表 t1 的所有数据话，策略很简单，就是分段放。
> 要是表 t1 是一个大表，join_buffer 放不下怎么办呢？join_buffer 的大小是由参数 join_buffer_size 设定的，默认值是 256k。如果放不下表 t1 的所有数据话，策略很简单，就是分段放。
> 1. 扫描表 t1，顺序读取数据行放入 join_buffer 中，放完第 88 行 join_buffer 满了，继续第 2 步；
> 2. 扫描表 t2，把 t2 中的每一行取出来，跟 join_buffer 中的数据做对比，满足 join 条件的，作为结果集的一部分返回；
> 3. 清空 join_buffer；
> 4. 继续扫描表 t1，顺序读取最后的 12 行数据放入 join_buffer 中，继续执行第 2 步
> 
执行流程图也就变成这样： ![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240303130853.png#id=BhqEu&originHeight=880&originWidth=1142&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
> 图中的步骤 4 和 5，表示清空 join_buffer 再复用。
> 这个流程才体现出了这个算法名字中“Block”的由来，表示“分块去 join”。
> 可以看到，这时候由于表 t1 被分成了两次放入 join_buffer 中，导致表 t2 会被扫描两次。虽然分成两次放入 join_buffer，但是判断等值条件的次数还是不变的，依然是 (88+12)*1000=10 万次。
> 我们再来看下，在这种情况下驱动表的选择问题。
> 假设，驱动表的数据行数是 N，需要分 K 段才能完成算法流程，被驱动表的数据行数是 M。注意，这里的 K 不是常数，N 越大 K 就会越大，因此把 K 表示为λ*N，显然λ的取值范围是 (0,1)。
> 所以，在这个算法的执行过程中：
> 1. 扫描行数是 N+λ_N_M；
> 2. 内存判断 N*M 次。
> 
显然，内存判断次数是不受选择哪个表作为驱动表影响的。而考虑到扫描行数，在 M 和 N 大小确定的情况下，N 小一些，整个算式的结果会更小。所以结论是，应该让小表当驱动表。

## 能不能使用 join 语句

1. 如果可以使用 Index Nested-Loop Join 算法，也就是说可以用上被驱动表上的索引，其实是没问题的；
2. 如果使用 Block Nested-Loop Join 算法，扫描行数就会过多。尤其是在大表上的 join 操作，这样可能要扫描被驱动表很多次，会占用大量的系统资源。所以这种 join 尽量不要用。
## join优化
如果要使用 join，应该选择大表做驱动表还是选择小表做驱动表？

1.  如果是 Index Nested-Loop Join 算法，应该选择小表做驱动表； 
2.  如果是 Block Nested-Loop Join 算法： 
   1. 在 join_buffer_size 足够大的时候，是一样的；
   2. 在 join_buffer_size 不够大的时候（这种情况更常见），应该选择小表做驱动表。

所以，这个问题的结论就是，总是应该使用小表做驱动表。
当然了，这里我需要说明下，什么叫作“小表”。
我们前面的例子是没有加条件的。如果我在语句的 where 条件加上 t2.id<=50 这个限定条件，再来看下这两条语句： 
```
select * from t1 straight_join t2 on (t1.b=t2.b) where t2.id<=50;
select * from t2 straight_join t1 on (t1.b=t2.b) where t2.id<=50;
```
注意，为了让两条语句的被驱动表都用不上索引，所以 join 字段都使用了没有索引的字段 b。
但如果是用第二个语句的话，join_buffer 只需要放入 t2 的前 50 行，显然是更好的。所以这里，“t2 的前 50 行”是那个相对小的表，也就是“小表”。
所以，更准确地说，在决定哪个表做驱动表的时候，应该是两个表按照各自的条件过滤，过滤完成之后，计算参与 join 的各个字段的总数据量，数据量小的那个表，就是“小表”，应该作为驱动表。
### Index Nested-Loop Join (NLJ) 算法的优化
为了便于分析，我还是创建两个表 t1、t2 来和你展开今天的问题。在表 t1 里，插入了 1000 行数据，每一行的 a=1001-id 的值。也就是说，表 t1 中字段 a 是逆序的。同时，在表 t2 中插入了 100 万行数据。
```
create table t1(id int primary key, a int, b int, index(a));
create table t2 like t1;
drop procedure idata;
delimiter ;;
create procedure idata()
begin
  declare i int;
  set i=1;
  while(i<=1000)do
    insert into t1 values(i, 1001-i, i);
    set i=i+1;
  end while;
  
  set i=1;
  while(i<=1000000)do
    insert into t2 values(i, i, i);
    set i=i+1;
  end while;

end;;
delimiter ;
call idata();
```
#### Multi-Range Read 优化
Multi-Range Read 优化 (MRR)。这个优化的主要目的是尽量使用顺序读盘。当非主键索引保存的是主键索引的key，所以当非主键索引命中之后，需要回表查询其他字段信息。回表是指，InnoDB 在普通索引 a 上查到主键 id 的值后，再根据一个个主键 id 的值到主键索引上去查整行数据的过程。
```
select * from t1 where a>=1 and a<=100;
```
主键索引是一棵 B+ 树，在这棵树上，每次只能根据一个主键 id 查到一行数据。因此，回表肯定是一行行搜索主键索引的，基本流程如图所示。
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240303131022.png#id=WTgGT&originHeight=880&originWidth=1142&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
如果随着 a 的值递增顺序查询的话，id 的值就变成随机的，那么就会出现随机访问，性能相对较差。虽然“按行查”这个机制不能改，但是调整查询的顺序，还是能够加速的。
因为大多数的数据都是按照主键递增顺序插入得到的，所以我们可以认为，如果按照主键的递增顺序查询的话，对磁盘的读比较接近顺序读，能够提升读性能。这就是 MRR 优化的设计思路。此时，语句的执行流程变成了这样：

1.  根据索引 a，定位到满足条件的记录，将 id 值放入 read_rnd_buffer 中 ; 
2.  将 read_rnd_buffer 中的 id 进行递增排序； 
3.  排序后的 id 数组，依次到主键 id 索引中查记录，并作为结果返回。
这里，read_rnd_buffer 的大小是由 read_rnd_buffer_size 参数控制的。如果步骤 1 中，read_rnd_buffer 放满了，就会先执行完步骤 2 和 3，然后清空 read_rnd_buffer。之后继续找索引 a 的下个记录，并继续循环。
另外需要说明的是，如果你想要稳定地使用 MRR 优化的话，需要设置set optimizer_switch="mrr_cost_based=off"。（官方文档的说法，是现在的优化器策略，判断消耗的时候，会更倾向于不使用 MRR，把 mrr_cost_based 设置为 off，就是固定使用 MRR 了。） 

![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240303131044.png#id=ihC8S&originHeight=880&originWidth=1142&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240303131103.png#id=DbkuB&originHeight=141&originWidth=1500&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
MRR 能够提升性能的核心在于，这条查询语句在索引 a 上做的是一个范围查询（也就是说，这是一个多值查询），可以得到足够多的主键 id。这样通过排序以后，再去主键索引查数据，才能体现出“顺序性”的优势。
Batched Key Access
理解了 MRR 性能提升的原理，我们就能理解 MySQL 在 5.6 版本后开始引入的 Batched Key Access(BKA) 算法了。这个 BKA 算法，其实就是对 NLJ 算法的优化。
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240303131203.png#id=xY9n2&originHeight=880&originWidth=1142&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
NLJ 算法执行的逻辑是：从驱动表 t1，一行行地取出 a 的值，再到被驱动表 t2 去做 join。也就是说，对于表 t2 来说，每次都是匹配一个值。这时，MRR 的优势就用不上了。
那怎么才能一次性地多传些值给表 t2 呢？方法就是，从表 t1 里一次性地多拿些行出来，一起传给表 t2。
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240303131229.png#id=Typui&originHeight=880&originWidth=1142&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
图中，我在 join_buffer 中放入的数据是 P1~P100，表示的是只会取查询需要的字段。当然，如果 join buffer 放不下 P1~P100 的所有数据，就会把这 100 行数据分成多段执行上图的流程。
如果要使用 BKA 优化算法的话，你需要在执行 SQL 语句之前，先设置
```
set optimizer_switch='mrr=on,mrr_cost_based=off,batched_key_access=on';
```
其中，前两个参数的作用是要启用 MRR。这么做的原因是，BKA 算法的优化要依赖于 MRR。
### Block Nested-Loop Join (BNL) 算法的优化
使用 Block Nested-Loop Join(BNL) 算法时，可能会对被驱动表做多次扫描。如果这个被驱动表是一个大的冷数据表，除了会导致 IO 压力大以外，还会对系统有什么影响呢？
由于 InnoDB 对 Bufffer Pool 的 LRU 算法做了优化，即：第一次从磁盘读入内存的数据页，会先放在 old 区域。如果 1 秒之后这个数据页不再被访问了，就不会被移动到 LRU 链表头部，这样对 Buffer Pool 的命中率影响就不大。
如果一个使用 BNL 算法的 join 语句，多次扫描一个冷表，而且这个语句执行时间超过 1 秒，就会在再次扫描冷表的时候，把冷表的数据页移到 LRU 链表头部。
这种情况对应的，是冷表的数据量小于整个 Buffer Pool 的 3/8，能够完全放入 old 区域的情况。
由于优化机制的存在，一个正常访问的数据页，要进入 young 区域，需要隔 1 秒后再次被访问到。但是，由于我们的 join 语句在循环读磁盘和淘汰内存页，进入 old 区域的数据页，很可能在 1 秒之内就被淘汰了。这样，就会导致这个 MySQL 实例的 Buffer Pool 在这段时间内，young 区域的数据页没有被合理地淘汰。
大表 join 操作虽然对 IO 有影响，但是在语句执行结束后，对 IO 的影响也就结束了。但是，对 Buffer Pool 的影响就是持续性的，需要依靠后续的查询请求慢慢恢复内存命中率。
为了减少这种影响，你可以考虑增大 join_buffer_size 的值，减少对被驱动表的扫描次数。也就是说，BNL 算法对系统的影响主要包括三个方面：

1.  可能会多次扫描被驱动表，占用磁盘 IO 资源； 
2.  判断 join 条件需要执行 M*N 次对比（M、N 分别是两张表的行数），如果是大表就会占用非常多的 CPU 资源； 
3.  可能会导致 Buffer Pool 的热数据被淘汰，影响内存命中率。
我们执行语句之前，需要通过理论分析和查看 explain 结果的方式，确认是否要使用 BNL 算法。如果确认优化器会使用 BNL 算法，就需要做优化。优化的常见做法是，给被驱动表的 join 字段加上索引，把 BNL 算法转成 BKA 算法。 
#### BNL 转 BKA
一些情况下，我们可以直接在被驱动表上建索引，这时就可以直接转成 BKA 算法了。但是，有时候你确实会碰到一些不适合在被驱动表上建索引的情况。比如下面这个语句：
```
select * from t1 join t2 on (t1.b=t2.b) where t2.b>=1 and t2.b<=2000;
```
我们在文章开始的时候，在表 t2 中插入了 100 万行数据，但是经过 where 条件过滤后，需要参与 join 的只有 2000 行数据。如果这条语句同时是一个低频的 SQL 语句，那么再为这个语句在表 t2 的字段 b 上创建一个索引就很浪费了。
但是，如果使用 BNL 算法来 join 的话，这个语句的执行流程是这样的：

1.  把表 t1 的所有字段取出来，存入 join_buffer 中。这个表只有 1000 行，join_buffer_size 默认值是 256k，可以完全存入。 
2.  扫描表 t2，取出每一行数据跟 join_buffer 中的数据进行对比 
   1. 如果不满足 t1.b=t2.b，则跳过；
   2. 如果满足 t1.b=t2.b, 再判断其他条件，也就是是否满足 t2.b 处于 [1,2000] 的条件，如果是，就作为结果集的一部分返回，否则跳过。

我在上一篇文章中说过，对于表 t2 的每一行，判断 join 是否满足的时候，都需要遍历 join_buffer 中的所有行。因此判断等值条件的次数是 1000*100 万 =10 亿次，这个判断的工作量很大。 
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240303131329.png#id=uaK9r&originHeight=143&originWidth=1500&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240303131347.png#id=XA4Sc&originHeight=130&originWidth=550&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
在表 t2 的字段 b 上创建索引会浪费资源，但是不创建索引的话这个语句的等值条件要判断 10 亿次，想想也是浪费。那么，有没有两全其美的办法呢？这时候，我们可以考虑使用临时表。

1. 把表 t2 中满足条件的数据放在临时表 tmp_t 中；
2. 为了让 join 使用 BKA 算法，给临时表 tmp_t 的字段 b 加上索引；
3. 让表 t1 和 tmp_t 做 join 操作。
```
create temporary table temp_t(id int primary key, a int, b int, index(b))engine=innodb;
insert into temp_t select * from t2 where b>=1 and b<=2000;
select * from t1 join temp_t on (t1.b=temp_t.b);
```
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240303131415.png#id=TXRLx&originHeight=547&originWidth=1130&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
总体来看，不论是在原表上加索引，还是用有索引的临时表，我们的思路都是让 join 语句能够用上被驱动表上的索引，来触发 BKA 算法，提升查询性能。
### 扩展 -hash join
如果 join_buffer 里面维护的不是一个无序数组，而是一个哈希表的话，那么就不是 10 亿次判断，而是 100 万次 hash 查找。这样的话，整条语句的执行速度就快多了吧？这，也正是 MySQL 的优化器和执行器一直被诟病的一个原因：不支持哈希 join。并且，MySQL 官方的 roadmap，也是迟迟没有把这个优化排上议程。
实际上，这个优化思路，我们可以自己实现在业务端。实现流程大致如下：

1.  select * from t1;取得表 t1 的全部 1000 行数据，在业务端存入一个 hash 结构，比如 C++ 里的 set、PHP 的数组这样的数据结构。 
2.  select * from t2 where b>=1 and b<=2000; 获取表 t2 中满足条件的 2000 行数据。 
3.  把这 2000 行数据，一行一行地取到业务端，到 hash 结构的数据表中寻找匹配的数据。满足匹配的条件的这行数据，就作为结果集的一行。
理论上，这个过程会比临时表方案的执行速度还要快一些。如果你感兴趣的话，可以自己验证一下。 
