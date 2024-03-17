# ElasticSearch 分页查询四种解决方案与原理

> 原文引用 -> [ElasticSearch分页查询四种解决方案与原理](https://blog.csdn.net/ztchun/article/details/91406928)

## 1、from + size 浅分页
常用的分页查询根据from+size语句如下：
```shell
GET /my_index/my_type/_search
{
    "query": { "match_all": {}},
    "from": 10,
    "size": 5
}
```
上面的查询表示从搜索结果中取第10条开始的5条数据。
这个查询语句在 Elasticsearch 集群内部是怎么执行？假设该索引只有primary shards，没有 replica shards，假设10个分片。搜索一般包括两个阶段，query 和 fetch 阶段，query 阶段确定要取哪些doc，fetch 阶段取出具体的 doc。
Query阶段
（1） Client 发送一次搜索请求，node1 接收到请求，然后，node1 创建一个大小为 from + size 的优先级队列用来存结果，我们管 node1 叫 coordinating node。
（2）coordinating node将请求广播到涉及到的 shards，每个 shard 在内部执行搜索请求，然后，将结果存到内部的大小同样为 from + size 的优先级队列里，可以把优先级队列理解为一个包含 top N 结果的列表。
（3）每个 shard 把暂存在自身优先级队列里的数据返回给 coordinating node，coordinating node 拿到各个 shards 返回的结果后对结果进行一次合并，产生一个全局的优先级队列，存到自身的优先级队列里。
在上面的过程中，coordinating node 拿到 (from + size) * 分片数目 条数据，然后合并并排序后选择前面的 from + size 条数据存到优先级队列，以便 fetch 阶段使用。另外，各个分片返回给 coordinating node 的数据用于选出前 from + size 条数据，所以，只需要返回唯一标记 doc 的 _id 以及用于排序的 _score 即可，这样也可以保证返回的数据量足够小。
coordinating node 计算好自己的优先级队列后，query 阶段结束，进入 fetch 阶段。
fetch阶段
query 阶段知道了要取哪些数据，但是并没有取具体的数据，这就是 fetch 阶段要做的。
（1）coordinating node 发送 GET 请求到相关shards。
（2）shard 根据 doc 的 _id 取到数据详情，然后返回给 coordinating node。
（3）coordinating node 返回数据给 Client。
coordinating node 的优先级队列里有 from + size 个 doc id，但是，在 fetch 阶段，并不需要取回所有数据，在上面的例子中，前10条数据是不需要取的，只需要取优先级队列里的第11到15条数据即可。
需要取的数据可能在不同分片，也可能在同一分片，coordinating node 使用 multi-get 来避免多次去同一分片取数据，从而提高性能。
## 2、scroll 深分页
from+size查询方式在10000-50000条数据（1000到5000页）以内的时候还是可以的，但是如果数据过多的话，就会出现深分页问题。
举例说明：
Elasticsearch 的这种方式提供了分页的功能，同时，也有相应的限制。举个例子，一个索引，有10亿数据，分10个 shards，然后，一个搜索请求，from=1,000,000，size=100，这时候，会带来严重的性能问题，CPU，内存，IO，网络带宽。
### 2.1 scroll
为了解决上面的问题，elasticsearch提出了一个scroll滚动的方式。
scroll 类似于sql中的cursor，使用scroll，每次只能获取一页的内容，然后会返回一个scroll_id。根据返回的这个scroll_id可以不断地获取下一页的内容，所以scroll并不适用于有跳页的情景。
(1)初始搜索请求应该在查询中指定 scroll 参数，如 ?scroll=1m,这可以告诉 Elasticsearch 需要保持搜索的上下文环境多久。
初始搜索：
```shell
GET /my_index/my_type/_search?scroll=1m
{
  "query": {
    "match_all": {}
  },
  "size": 1,
  "from": 0
}
```
返回结果：
```shell
{
  "_scroll_id": "DnF1ZXJ5VGhlbkZldGNoBQAAAAAAABn5FjRXNkZmY3ZmVGJPVXJ1NWs2MUh5RGcAAAAAAAAZ_BY0VzZGZmN2ZlRiT1VydTVrNjFIeURnAAAAAAAAGfgWNFc2RmZjdmZUYk9VcnU1azYxSHlEZwAAAAAAABn6FjRXNkZmY3ZmVGJPVXJ1NWs2MUh5RGcAAAAAAAAZ-xY0VzZGZmN2ZlRiT1VydTVrNjFIeURn",
  "took": 0,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
  }
```
返回结果包含一个 scroll_id，可以被传递给 scroll API 来检索下一个批次的结果。
（2）每次对 scroll API 的调用返回了结果的下一个批次结果，直到 hits 数组为空。scroll_id 则可以在请求体中传递。scroll 参数告诉 Elasticsearch 保持搜索的上下文等待另一个3m。返回数据的size与初次请求一致。
二次搜索：
```shell
POST /_search/scroll
{
  "scroll":"3m",
  "scroll_id": "DnF1ZXJ5VGhlbkZldGNoBQAAAAAAAB0mFjRXNkZmY3ZmVGJPVXJ1NWs2MUh5RGcAAAAAAAAdKBY0VzZGZmN2ZlRiT1VydTVrNjFIeURnAAAAAAAAHScWNFc2RmZjdmZUYk9VcnU1azYxSHlEZwAAAAAAAB0qFjRXNkZmY3ZmVGJPVXJ1NWs2MUh5RGcAAAAAAAAdKRY0VzZGZmN2ZlRiT1VydTVrNjFIeURn"
}
```
返回结果：
```shell
{
  "_scroll_id": "DnF1ZXJ5VGhlbkZldGNoBQAAAAAAAB0mFjRXNkZmY3ZmVGJPVXJ1NWs2MUh5RGcAAAAAAAAdKBY0VzZGZmN2ZlRiT1VydTVrNjFIeURnAAAAAAAAHScWNFc2RmZjdmZUYk9VcnU1azYxSHlEZwAAAAAAAB0qFjRXNkZmY3ZmVGJPVXJ1NWs2MUh5RGcAAAAAAAAdKRY0VzZGZmN2ZlRiT1VydTVrNjFIeURn",
  "took": 1,
  "timed_out": false,
  "terminated_early": true,
  "_shards": {
    "total": 5,
    "successful": 5,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": 5168,
    "max_score": 1,
    "hits": [
      {
      }
```
原理上来说可以把 scroll 分为初始化和遍历两步，初始化时将所有符合搜索条件的搜索结果缓存起来，可以想象成快照，在遍历时，从这个快照里取数据，也就是说，在初始化后对索引插入、删除、更新数据都不会影响遍历结果。因此，scroll 并不适合用来做实时搜索，而更适用于后台批处理任务，比如群发。
### 2.2 scroll-scan 的高效滚动
scroll API 保持了那些已经返回记录结果，所以能更加高效地返回排序的结果。但是，按照默认设定排序结果仍然需要代价。
一般来说，你仅仅想要找到结果，不关心顺序。你可以通过组合 scroll 和 scan 来关闭任何打分或者排序，以最高效的方式返回结果。你需要做的就是将 search_type=scan 加入到查询的字符串中：
```shell
POST /my_index/my_type/_search?scroll=1m&search_type=scan
{
   "query": {
        "match" : {
            "cityName" : "杭州"
        }
    }
}
```
设置 search_type 为 scan 可以关闭打分，让滚动更加高效。
扫描式的滚动请求和标准的滚动请求有四处不同：
(1)不算分，关闭排序。结果会按照在索引中出现的顺序返回;
(2)不支持聚合;
(3)初始 search 请求的响应不会在 hits 数组中包含任何结果。第一批结果就会按照第一个 scroll 请求返回。
(4)参数 size 控制了每个分片上而非每个请求的结果数目，所以 size 为 10 的情况下，如果命中了 5 个分片，那么每个 scroll 请求最多会返回 50 个结果。
如果你想支持打分，即使不进行排序，将 track_scores 设置为 true。
## 3、search_after 深分页
scroll 的方式，官方的建议不用于实时的请求（一般用于数据导出），因为每一个scroll_id 不仅会占用大量的资源，而且会生成历史快照，对于数据的变更不会反映到快照上。
search_after 分页的方式是根据上一页的最后一条数据来确定下一页的位置，同时在分页请求的过程中，如果有索引数据的增删改查，这些变更也会实时的反映到游标上。但是需要注意，因为每一页的数据依赖于上一页最后一条数据，所以无法跳页请求。
为了找到每一页最后一条数据，每个文档必须有一个全局唯一值，官方推荐使用 _uid 作为全局唯一值，其实使用业务层的 id 也可以。
（1）首次查询
```shell
POST /my_index/my_type/_search
{
    "size":2,
    "query": {
        "match" : {
            "cityName" : "杭州"
        }
    },
    "sort": [
        {"updateTime": "desc"}
    ]
}
```
查询返回结果
```shell
{
  "took": 2,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": 534,
    "max_score": null,
    "hits": [
      {
        "_index": "my_index",
        "_type": "my_type",
        "_id": "2019061010810316",
        "_score": null,
        "_source": {
        }
```
（2）第二次查询
```shell
POST /my_index/my_type/_search{    "size":2,    "query": {        "match" : {            "cityName" : "杭州"        }    },    "search_after": [1560137241000],    "sort": [        {"updateTime": "desc"}    ]}
```
查询结果：
按照第一个检索到的最后显示的“updateTime”，search_after及多个排序字段多个参数用逗号隔开，作为下一个检索search_after的参数。
当使用search_after参数时，from的值必须被设为0或者-1
```shell
{  "took": 5,  "timed_out": false,  "_shards": {    "total": 5,    "successful": 5,    "skipped": 0,    "failed": 0  },  "hits": {    "total": 534,    "max_score": null,    "hits": [      {        "_index": "my_index",        "_type": "my_type",        "_id": "2019061010781031",        "_score": null,        "_source": {        }
```
