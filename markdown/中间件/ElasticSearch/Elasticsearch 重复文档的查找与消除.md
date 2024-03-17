# Elasticsearch 重复文档的查找与消除

令人惊讶的是，关于此主题的文档很少，而且几乎都出自我引用的这篇博文：[消除 Elasticsearch 中的重复文档](https://qbox.io/blog/minimizing-document-duplication-in-elasticsearch)
避免 Elasticsearch 索引中的重复总是一件好事。但是您可以通过消除重复获得其他好处：节省磁盘空间、提高搜索准确性、提高硬件资源管理效率。也许最重要的是，您减少了搜索的获取时间。
## 示例数据
这里有四个简单的文档，其中一个是另一个的副本。我们在 name`employeeid`和 type下索引这些文档`info`。
### 文档1
```shell
curl -XPOST 'http://localhost:9200/employeeid/info/1' -d '{
 "name": "John",
 "organisation": "Apple",
 "employeeID": "23141A"
}'
```
### 文档2
```shell
curl -XPOST 'http://localhost:9200/employeeid/info/2' -d '{
 "name": "Sam",
 "organisation": "Tesla",
 "employeeID": "TE9829"
 }'
```
### 文档3
```shell
curl -XPOST 'http://localhost:9200/employeeid/info/3' -d '{
 "name":"Sarah",
 "organisation":"Microsoft",
 "employeeID" :"M54667"
 }'
```
### 文档4
```shell
curl -XPOST 'http://localhost:9200/employeeid/info/4' -d '{
 "name": "John",
 "organisation": "Apple",
 "employeeID": "23141A"
 }'
```
仔细观察，您会发现文档 4 是文档 1 的副本。
## 在索引期间避免重复文档
在我们考虑如何在 Elasticsearch 中执行重复检查之前，让我们花点时间考虑一下不同类型的索引场景。
一种情况是我们可以在索引之前访问源文档。在这种情况下，检查数据并找到一个或多个包含唯一值的字段相对容易。也就是说，该字段的每个不同值都恰好出现在一个文档中。在这种情况下，我们可以将该特定字段设置为 Elasticsearch 索引的文档 ID。由于任何重复的源文档也将具有相同的文档 ID，因此 Elasticsearch 将确保这些重复项不会成为索引的一部分。
## 向上插入
另一种情况是当一个或多个文档具有相同的标识符但内容不同时。当用户编辑文档并希望使用相同的文档 ID 重新索引该文档时，通常会发生这种情况。问题是当用户尝试重新索引时，Elasticsearch 不会允许它，因为它的文档 ID 必须是唯一的。
解决方法是使用[upsert API](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-update.html#upserts)。Upsert检查特定文档是否存在，如果存在，upsert 将使用 upsert 的内容更新该文档。如果文档不存在，upsert 将创建具有相同内容的文档。无论哪种方式，用户都会在相同的文档 ID 下获得内容更新。
在第三种情况下，在创建索引之前无法访问数据集。在这些情况下，我们需要搜索索引并检查重复项。这就是我们在以下部分中演示的内容。
## 重复的基本检查
在我们的每个示例文档中，我们看到三个字段：`name, organisation,`如果我们首先假设该字段`name`是唯一的，则指定该字段作为检查重复项的标识符。如果多个文档具有相同的`name`字段值，则该文档确实是重复的。
考虑到这个基本原理，我们可以执行简单的[术语聚合]([https://www.elastic.co/guide/en/elasticsearch/reference/current/search-aggregations-bucket-terms-aggregation.html?q=terms](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-aggregations-bucket-terms-aggregation.html?q=terms) ag)来获取 field 的每个值的文档计数`name`。但是，这种简单的聚合只会返回该字段的每个值下的文档计数。这种方法在检查重复项时没有用，因为我们要检查文档中字段的一个或多个值的重复项。为此，我们还需要应用[top_hits 聚合]([https://www.elastic.co/guide/en/elasticsearch/reference/current/search-aggregations-metrics-top-hits-aggregation.html?q=top_hits](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-aggregations-metrics-top-hits-aggregation.html?q=top_hits) aggregation)——一个子[聚合器]([https://www.elastic.co/guide/en/elasticsearch/reference/current/search-aggregations-metrics-top-hits-aggregation.html?q=top_hits](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-aggregations-metrics-top-hits-aggregation.html?q=top_hits) aggregation)，其中每个桶聚合顶部匹配的文档。
以下是我们针对上面给出的示例文档索引推荐的查询：
```shell
curl -XGET 'http://localhost:9200/employeeid/info/_search?pretty=true' -d '{
  "size": 0,
  "aggs": {
    "duplicateCount": {
      "terms": {
      "field": "name",
        "min_doc_count": 2
      },
      "aggs": {
        "duplicateDocuments": {
          "top_hits": {}
        }
      }
    }
  }
}'
```
这里我们定义了参数`min_doc_count`。通过将此参数设置为 2，只有 a`doc_count`为 2 或更多的聚合桶才会出现在聚合中（如下面的结果所示）。
```json
{
  "took": 112,
  "timed_out": false,
  "_shards": {
    "total": 4,
    "successful": 4,
    "failed": 0
  },
  "hits": {
    "total": 4,
    "max_score": 0,
    "hits": [
    ]
  },
  "aggregations": {
    "duplicateCount": {
      "doc_count_error_upper_bound": 0,
      "sum_other_doc_count": 0,
      "buckets": [
        {
          "key": "john",
          "doc_count": 2,
          "duplicateDocuments": {
            "hits": {
              "total": 2,
              "max_score": 1,
              "hits": [
                {
                  "_index": "employeeid",
                  "_type": "info",
                  "_id": "4'",
                  "_score": 1,
                  "_source": {
                    "name": "John",
                    "organisation": "Apple",
                    "employeeID": "23141A"
                  }
                },
                {
                  "_index": "employeeid",
                  "_type": "info",
                  "_id": "1'",
                  "_score": 1,
                  "_source": {
                    "name": "John",
                    "organisation": "Apple",
                    "employeeID": "23141A"
                  }
                }
              ]
            }
          }
        }
      ]
    }
  }
}
```
需要注意的是，我们必须将 的值设置`min_doc_count`为 2。否则，其他结果将出现在聚合中，我们将找不到任何可能存在的重复项。
## 对多个字段中的值进行重复数据删除
我们上面所做的是根据单个字段中的值识别重复文档的一个非常基本的示例。这不是很有趣。或者有用。在大多数情况下，检查重复项需要检查多个字段。我们不能可靠地假设员工文档中存在重复项，这些文档仅包含该`name`字段中多次出现的“Bill”值。在许多实际情况下，有必要检查许多不同领域的重复。考虑到我们上面的示例数据集，我们需要检查所有字段中的重复。
我们可以从上一节扩展我们的方法，并执行多字段术语聚合和热门聚合。我们可以对索引文档中的所有三个字段进行术语聚合。我们将再次指定`min_doc_count`参数以仅获取 a`doc_count`大于或等于 2 的桶。我们还应用`top_hits`聚合来获得正确的结果。为了容纳多个字段，我们使用了一个脚本来帮助我们附加字段值以在聚合中显示：
```shell
curl -XGET 'http://localhost:9200/employeeid/info/_search?pretty=true' -d '{
  "size": 0,
  "aggs": {
    "duplicateCount": {"terms": {
      "script": "doc['name'].values + doc['employeeID'].values+doc['organisation'].values",
      "min_doc_count": 2
    },      
    "aggs": {}
      "duplicateDocuments": {
        "top_hits": {}
      }
    }
  }
}'
```
如下所示，运行此查询的结果显示了一个`duplicateCount`聚合，其中我们总共得到三个键值——每个`doc_count`值都是 2。同样在每个键值下，聚合`duplicateDocuments`包含发现重复值的文档。我们可以对这些文件进行交叉检查和验证。
```json
{
  "took": 9,
  "timed_out": false,
  "_shards": {
    "total": 4,
    "successful": 4,
    "failed": 0
  },
  "hits": {
    "total": 4,
    "max_score": 0,
    "hits": [
    ]
  },
  "aggregations": {
    "duplicateCount": {
      "doc_count_error_upper_bound": 0,
      "sum_other_doc_count": 0,
      "buckets": [
        {
          "key": "23141a",
          "doc_count": 2,
          "duplicateDocuments": {
            "hits": {
              "total": 2,
              "max_score": 1,
              "hits": [
                {
                  "_index": "employeeid",
                  "_type": "info",
                  "_id": "4'",
                  "_score": 1,
                  "_source": {
                    "name": "John",
                    "organisation": "Apple",
                    "employeeID": "23141A"
                  }
                },
                {
                  "_index": "employeeid",
                  "_type": "info",
                  "_id": "1'",
                  "_score": 1,
                  "_source": {
                    "name": "John",
                    "organisation": "Apple",
                    "employeeID": "23141A"
                  }
                }
              ]
            }
          }
        },
        {
          "key": "apple",
          "doc_count": 2,
          "duplicateDocuments": {
            "hits": {
              "total": 2,
              "max_score": 1,
              "hits": [
                {
                  "_index": "employeeid",
                  "_type": "info",
                  "_id": "4'",
                  "_score": 1,
                  "_source": {
                    "name": "John",
                    "organisation": "Apple",
                    "employeeID": "23141A"
                  }
                },
                {
                  "_index": "employeeid",
                  "_type": "info",
                  "_id": "1'",
                  "_score": 1,
                  "_source": {
                    "name": "John",
                    "organisation": "Apple",
                    "employeeID": "23141A"
                  }
                }
              ]
            }
          }
        },
        {
          "key": "john",
          "doc_count": 2,
          "duplicateDocuments": {
            "hits": {
              "total": 2,
              "max_score": 1,
              "hits": [
                {
                  "_index": "employeeid",
                  "_type": "info",
                  "_id": "4'",
                  "_score": 1,
                  "_source": {
                    "name": "John",
                    "organisation": "Apple",
                    "employeeID": "23141A"
                  }
                },
                {
                  "_index": "employeeid",
                  "_type": "info",
                  "_id": "1'",
                  "_score": 1,
                  "_source": {
                    "name": "John",
                    "organisation": "Apple",
                    "employeeID": "23141A"
                  }
                }
              ]
            }
          }
        }
      ]
    }
  }
}
```
