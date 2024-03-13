# ElasticSearch 复合查询 Compound queries

## 布尔查询 Boolean query
一种查询，用于匹配与其他查询的布尔组合相匹配的文档。bool查询映射到Lucene BooleanQuery。它是用一个或多个布尔子句构建的，每个子句都有一个类型化的出现。出现类型有:

| 类型 | 描述 |
| --- | --- |
| `must` | 子句(查询)必须出现在匹配的文档中，并将有助于得分。 |
| `filter` | 子句(查询)必须出现在匹配的文档中。然而，与must不同的是，查询的分数将被忽略。过滤器子句在过滤器上下文中执行，这意味着评分被忽略，子句被考虑用于缓存。 |
| `should` | 子句(查询)应该出现在匹配的文档中。 |
| `must_not` | 子句(查询)不能出现在匹配的文档中。子句在过滤器上下文中执行，这意味着评分被忽略，子句被考虑用于缓存。因为评分会被忽略，所以会返回所有文档的评分为0。 |

bool查询采用“越匹配越好”的方法，因此每个匹配的must或should子句的分数将被添加到一起，以提供每个文档的最终_score。
```
POST _search
{
  "query": {
    "bool" : {
      "must" : {
        "term" : { "user.id" : "kimchy" }
      },
      "filter": {
        "term" : { "tags" : "production" }
      },
      "must_not" : {
        "range" : {
          "age" : { "gte" : 10, "lte" : 20 }
        }
      },
      "should" : [
        { "term" : { "tags" : "env1" } },
        { "term" : { "tags" : "deployed" } }
      ],
      "minimum_should_match" : 1,
      "boost" : 1.0
    }
  }
}
```

-  minimum_should_match 的使用 
   - 你可以使用' minimum_should_match '参数来指定返回的文档_must_匹配的' should '子句的数量或百分比。
   - 如果' bool '查询至少包含一个' should '子句且没有' must '或' filter '子句，则默认值为' 1 '。否则，默认值为“0”。
-  得分 与 bool.filteredit 
   -  在过滤器元素下指定的查询对得分没有影响——得分返回为0。分数只受指定查询的影响。例如，以下三个查询都返回状态字段包含术语active的所有文档。 
      -  第一个查询将所有文档的得分赋为0，因为没有指定得分查询: 
```
GET _search
{
  "query": {
    "bool": {
      "filter": {
        "term": {
          "status": "active"
        }
      }
    }
  }
}
```

      -  这个bool查询有一个match_all查询，它为所有文档分配1.0的分数。 
```
GET _search
{
  "query": {
    "bool": {
      "must": {
        "match_all": {}
      },
      "filter": {
        "term": {
          "status": "active"
        }
      }
    }
  }
}
```

      -  这个constant_score查询的行为与上面的第二个例子完全相同。constant_score查询为过滤器匹配的所有文档分配一个1.0的分数。 
```
GET _search
{
  "query": {
    "constant_score": {
      "filter": {
        "term": {
          "status": "active"
        }
      }
    }
  }
}
```
## 增强查询 Boosting query
返回与正查询匹配的文档，同时降低与负查询匹配的文档的相关性评分。
您可以使用提升查询来降级某些文档，而不将它们从搜索结果中排除。
```
GET /_search
{
  "query": {
    "boosting": {
      "positive": {
        "term": {
          "text": "apple"
        }
      },
      "negative": {
        "term": {
          "text": "pie tart fruit crumble tree"
        }
      },
      "negative_boost": 0.5
    }
  }
}
```
| 顶级参数 | 描述 |
| --- | --- |
| **positive** | (必需的，查询对象)希望运行的查询。任何返回的文档都必须与此查询匹配。 |
| **negative** | (必需，查询对象)用于降低匹配文档的相关度的查询。
如果返回的文档与正查询匹配，则boost查询计算文档的最终关联得分，如下所示:
1. 从正面查询中获取原始的相关性得分。
2. 将分数乘以negative_boost值。 |
| **negative_boost** | (必需，浮点数)0到1.0之间的浮点数，用于降低匹配负查询的文档的相关性得分。 |

## 常数分数查询 Constant score query
包装一个过滤器查询，并使用等于boost参数值的相关分数返回每个匹配的文档。
```
GET /_search
{
  "query": {
    "constant_score": {
      "filter": {
        "term": { "user.id": "kimchy" }
      },
      "boost": 1.2
    }
  }
}
```
| 顶级参数 | 描述 |
| --- | --- |
| **filter** | (必需的，查询对象)要运行的过滤查询。任何返回的文档都必须与此查询匹配。
过滤查询不计算关联分数。为了提高性能，Elasticsearch会自动缓存经常使用的过滤器查询。 |
| **boost** | (可选，浮点数)浮点数用作每个匹配过滤器查询的文档的固定相关性得分。默认为1.0。 |

