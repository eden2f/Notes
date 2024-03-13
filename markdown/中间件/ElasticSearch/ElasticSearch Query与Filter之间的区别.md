# ElasticSearch Query与Filter之间的区别

> [JavaShuo : Elasticsearch中Query与Filter之间的区别](http://www.javashuo.com/article/p-vskgdmmx-he.html)
[官方文档 : Query and filter context](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-filter-context.html)

## 查询器与过滤器缓存
尽管咱们以前已经涉及了查询DSL，然而实际上存在两种DSL：查询DSL（query DSL）和过滤DSL（filter DSL）。而查询子句（query clause）和过滤器子句（filter clause）实际上也相似，只是它们的目的稍微不一样。
### 过滤器（filter）
过滤器（filter）用于向全部文档提问是非题，一般用于过滤文档的某个字段是否包含某个准确的值:

- 建立日期是否在2013-2014年间？
- status字段是否为published？
- lat_lon字段是否在某个坐标的10千米范围内？
### 查询器（query）
查询器（query）很像过滤器，也是用于提问：这个文档有多匹配？
查询器的典型用法是用于查找文档：

- 与_full text search_的匹配度最高文档
- 包含_run_单词，若是包含这些单词：runs、running、jog、sprint，也被视为包含
- 包含quick、brown、fox。这些词越接近，这份文档的相关性就越高

查询器会计算出每份文档对于某次查询有多相关（relevant），而后分配文档一个相关性分数：_score。而这个分数会被用来对匹配了的文档进行相关性排序。相关性概念十分适合全文搜索（full-text search），这个很难能给出完整、“正确”答案的领域。
## 性能上的不一样
大多数过滤器子句的输出——一个经过筛选的文档列表——能够被快速的被计算出，同时能够被轻易的缓存在内存中，并且每份文档只用1比特。这些被缓存了的过滤器能够被后续的请求有效利用。
而查询器不仅是要找出匹配的文档，同时还要计算出每份文档的相关性。因此一般查询器比过滤器更重，并且查询结果是不可缓存的。
感谢反向索引（inverted index）使用得一个只匹配少许文档的简单查询的性能能够媲美甚至超过一个缓存了跨了千万份文档的过滤器。然而，一个缓存了的过滤器的性能一般会赛过查询器，并且一直都将是这样。
使用过滤器的目的是精简查询器查询后结果的文档数。
