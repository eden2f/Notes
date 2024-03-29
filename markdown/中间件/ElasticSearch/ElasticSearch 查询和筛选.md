# ElasticSearch 查询和筛选

## 前言
查询子句的行为有两种,过滤 和 查询. 在查询上下文中使用查询子句来处理影响匹配文档得分的条件(例如，文档匹配的程度)，并在过滤上下文中使用所有其他查询子句。
## 查询器 Query context
在查询上下文中使用的查询子句可以回答这样一个问题:“这个文档与这个查询子句的匹配程度如何?”除了决定文档是否匹配外，查询子句还计算一个_score，表示文档相对于其他文档的匹配程度。
查询上下文在将查询子句传递给查询参数(例如搜索API中的查询参数)时生效。
## 筛选器 Filter context
在过滤器上下文中，查询子句回答“此文档是否匹配此查询子句?”答案是简单的“是”或“不是”——不计算分数。过滤上下文主要用于过滤结构化数据，例如。

-  这个时间戳是否属于2015到2016的范围? 
-  状态字段是否设置为“published”? 

Elasticsearch会自动缓存常用的筛选器，以提高性能。
只要将查询子句传递给筛选器参数，例如bool查询中的Filter或must_not参数、constant_score查询中的Filter参数或筛选器聚合，筛选器上下文就会生效。
## 查询子句详解
下面是在搜索API的查询和过滤上下文中使用的查询子句的示例。该查询将匹配满足以下所有条件的文档:

-  标题字段包含单词搜索。 
-  内容字段包含单词elasticsearch。 
-  状态字段包含发布的确切单词。 
-  publish_date 字段包含从2015年1月1日起的日期。 
```
GET /_search
{
  "query": { 
    "bool": { 
      "must": [
        { "match": { "title":   "Search"        }}, 
        { "match": { "content": "Elasticsearch" }}  
      ],
      "filter": [ 
        { "term":  { "status": "published" }}, 
        { "range": { "publish_date": { "gte": "2015-01-01" }}} 
      ]
    }
  }
}
```

-  **query** 参数表示查询上下文。 
-  在查询上下文中使用 **bool** 和两个 **match** 子句，这意味着它们用于对每个文档的匹配程度进行评分。 
-  **filter**参数表示过滤上下文。 
-  **term** 和 **range** 子句用于筛选器上下文中。它们将过滤出不匹配的文档，但不会影响匹配文档的得分。 
