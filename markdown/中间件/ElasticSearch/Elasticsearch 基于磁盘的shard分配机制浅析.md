# Elasticsearch 基于磁盘的shard分配机制浅析

> 原文引用 : [http://niyanchun.com/es-disk-based-shard-allocation.html](http://niyanchun.com/es-disk-based-shard-allocation.html)

先回顾几个概念：ES的Index是个逻辑概念，实际由若干shard组成，而shard就是Lucene的Index，即真正存储数据的实体。当有数据需要存储的时候，就需要先分配shard。具体来说需要分配shard的场景包括：数据恢复，主分片（primary）、副本分片的分配，再平衡（rebalancing），节点的新增、删除。对于分布式存储系统来说，数据的分布非常重要，ES shard的分配工作由ES的master节点负责。
ES提供了多种分配策略的支持，简单来说就是用户可以通过配置定义一些“策略”或者叫“路由规则”，然后ES会在遵守这些策略的前提下，尽可能的让数据均匀分布。比如可以配置机房、机架属性，ES会尽量让主数据和副本数据分配在不同的机房、机架，起到容灾的作用。再比如，可以配置一些策略，让数据不分配到某些节点上面，这在滚动升级或者数据迁移的时候非常有用。不过本文并不会介绍所有这些策略，只聚焦于默认的基于磁盘的分配策略，因为这部分是最常用的。
先说一下再平衡。
## 再平衡
再平衡这个概念在分布式存储系统里面很常见，几乎是标配。因为各种各样的原因（比如分配策略不够智能、新增节点等），系统各个节点的数据存储可能分布不均，这时候就需要有能够重新让数据均衡的机制，即所谓的再平衡。有些系统的再平衡需要用户手动执行，有些则是自动的。ES就属于后者，它的再平衡是自动的，用户不参与，也几乎不感知。
那到底怎样才算平衡（balanced）？ES官方对此的定义是：
> A cluster is balanced when it has an equal number of shards on each node without having a concentration of shards from any index on any node.

简单说就是看每个节点上面的shard个数是否相等，越相近就越平衡。这里注意计数的时候是“无差别”的，即不管是哪个索引的shard，也不管是主分片的shard，还是副本的shard，一视同仁，只算个数。ES后台会有进程专门检查整个集群是否平衡，以及执行再平衡的操作。再平衡也就是将shard数多的迁移到shard数少的节点，让集群尽可能的平衡。
关于再平衡，需要注意2个点：
平衡的状态是一个范围，而不是一个点。即不是说各个节点的shard数严格相等才算平衡，而是大家的差别在一个可接受的范围内就算平衡。这个范围（也称阈值或权重）是可配置的，用户一般是无需参与的。
再平衡是一个尽力而为的动作，它会在遵守各种策略的前提下，尽量让集群趋于平衡。
看个简单的例子吧。有一个集群刚开始有2个节点（node42，node43），我们创建一个1 replica、6 primary shard的索引shard_alloc_test：
```
PUT shard_alloc_test
{
  "settings": {
      "index": {   
          "number_of_shards" : "6",          
          "number_of_replicas": "1"
      }
  }   
}
```
查看一下shard的分配：
```
GET _cat/shards?v
index                               shard prirep state   docs  store ip        node
shard_alloc_test                    3     r      STARTED    0   208b 10.8.4.43 node-43
shard_alloc_test                    3     p      STARTED    0   208b 10.8.4.42 node-42
shard_alloc_test                    2     p      STARTED    0   208b 10.8.4.43 node-43
shard_alloc_test                    2     r      STARTED    0   208b 10.8.4.42 node-42
shard_alloc_test                    4     p      STARTED    0   208b 10.8.4.43 node-43
shard_alloc_test                    4     r      STARTED    0   208b 10.8.4.42 node-42
shard_alloc_test                    1     r      STARTED    0   208b 10.8.4.43 node-43
shard_alloc_test                    1     p      STARTED    0   208b 10.8.4.42 node-42
shard_alloc_test                    5     r      STARTED    0   208b 10.8.4.43 node-43
shard_alloc_test                    5     p      STARTED    0   208b 10.8.4.42 node-42
shard_alloc_test                    0     p      STARTED    0   208b 10.8.4.43 node-43
shard_alloc_test                    0     r      STARTED    0   208b 10.8.4.42 node-42
```
可以看到，primary的6个shard和replica的6个shard的分配是非常均衡的：一方面，12个shard均匀分配到了2个节点上面；另一方面，primary shard和replica shard也是均匀交叉的分配到了2个节点上面。
此时，我们对集群进行扩容，再增加一台节点：node-41。待节点成功加入集群后，我们看一下shard_alloc_test的shard分配：
```
GET _cat/shards?v
index                               shard prirep state   docs  store ip        node
shard_alloc_test                    3     p      STARTED    0   208b 10.8.4.41 node-41
shard_alloc_test                    3     r      STARTED    0   208b 10.8.4.43 node-43
shard_alloc_test                    2     p      STARTED    0   208b 10.8.4.41 node-41
shard_alloc_test                    2     r      STARTED    0   208b 10.8.4.42 node-42
shard_alloc_test                    4     p      STARTED    0   208b 10.8.4.41 node-41
shard_alloc_test                    4     r      STARTED    0   208b 10.8.4.42 node-42
shard_alloc_test                    1     r      STARTED    0   208b 10.8.4.43 node-43
shard_alloc_test                    1     p      STARTED    0   208b 10.8.4.42 node-42
shard_alloc_test                    5     p      STARTED    0   208b 10.8.4.41 node-41
shard_alloc_test                    5     r      STARTED    0   208b 10.8.4.43 node-43
shard_alloc_test                    0     p      STARTED    0   208b 10.8.4.43 node-43
shard_alloc_test                    0     r      STARTED    0   208b 10.8.4.42 node-42


GET _cat/tasks?v
action                         task_id                       parent_task_id              type      start_time    timestamp running_time ip        node
indices:data/read/get[s]       0KbNzeuGR1iZ1I1_3fIDVg:192304 -                           transport 1622513252244 02:07:32  1.6ms        10.8.4.42 node-42
indices:data/read/get          jgfZk3snRb-uEUToQgo9pw:1841   -                           transport 1622513252487 02:07:32  1.4ms        10.8.4.43 node-43
```
可以看到，shard自动的进行了再分配，均匀的分配到了3个节点上面。如果再平衡的时间稍长一点，你还可以通过task接口看到集群间的数据迁移任务。然后我们再缩容，停掉node-41这个节点，数据也会再次自动重新分配：
```
GET _cat/shards?v
index                               shard prirep state   docs  store ip        node
shard_alloc_test                    3     p      STARTED    0   208b 10.8.4.43 node-43
shard_alloc_test                    3     r      STARTED    0   208b 10.8.4.42 node-42
shard_alloc_test                    2     r      STARTED    0   208b 10.8.4.43 node-43
shard_alloc_test                    2     p      STARTED    0   208b 10.8.4.42 node-42
shard_alloc_test                    1     r      STARTED    0   208b 10.8.4.43 node-43
shard_alloc_test                    1     p      STARTED    0   208b 10.8.4.42 node-42
shard_alloc_test                    4     r      STARTED    0   208b 10.8.4.43 node-43
shard_alloc_test                    4     p      STARTED    0   208b 10.8.4.42 node-42
shard_alloc_test                    5     p      STARTED    0   208b 10.8.4.43 node-43
shard_alloc_test                    5     r      STARTED    0   208b 10.8.4.42 node-42
shard_alloc_test                    0     p      STARTED    0   208b 10.8.4.43 node-43
shard_alloc_test                    0     r      STARTED    0   208b 10.8.4.42 node-42
```
所以，ES的再平衡功能还是非常好用和易用的，完全自动化。但是细心的同学应该已经意识到一个问题：光靠保证shard个数均衡其实是没法保证数据均衡的，因为有些shard可能很大，存了很多数据；有些shard可能很小，只存了几条数据。的确是这样，所以光靠再平衡还是无法保证数据的均衡的，至少从存储容量的角度来说是不能保证均衡的。所以，ES还有一个默认就开启的基于磁盘容量的shard分配器。
## 基于磁盘容量的shard分配
基于磁盘容量的shard分配策略（Disk-based shard allocation）默认就是开启的，其机制也非常简单，主要就是3条非常重要的分水线（watermark）：
low watermark：默认值是85%。磁盘使用超过这个阈值，就认为“危险”快来了，这个时候就不会往该节点再分配replica shard了，但新创建的索引的primary shard还是可以分配。特别注意必须是新创建的索引（什么是“老的”？比如再平衡时其它节点上已经存在的primary shard就算老的，这部分也是不能够迁移到进入low watermark的节点上来的）。
high watermark：默认值是90%。磁盘使用超过这个阈值，就认为“危险”已经来了，这个时候不会再往该节点分配任何shard，即primary shard和replica shard都不会分配。并且会开始尝试将节点上的shard迁移到其它节点上。
flood stage watermark：默认值是95%。磁盘使用超过这个阈值，就认为已经病入膏肓了，需要做最后的挽救了，挽救方式也很简单——断臂求生：将有在该节点上分配shard的所有索引设置为只读，不允许再往这些索引写数据，但允许删除索引（index.blocks.read_only_allow_delete）。
大概总结一下：

- 当进入low watermark的时候，就放弃新创建的索引的副本分片数据了（即不创建对应的shard），但还是允许创建主分片数据；
- 当进入high watermark的时候，新创建索引的主分片、副本分片全部放弃了，但之前已经创建的索引还是可以正常继续写入数据的；同时尝试将节点上的数据向其它节点迁移；
- 当进入flood stage watermark，完全不允许往该节点上写入数据了，这是最后一道保护。只要在high watermark阶段，数据可以迁移到其它节点，并且迁移的速度比写入的速度快，那就不会进入该阶段。

一些相关的配置如下：

- cluster.routing.allocation.disk.threshold_enabled：是否开启基于磁盘的分配策略，默认为true，表示开启。
- cluster.info.update.interval：多久检查一次磁盘使用，默认值是30s。
- cluster.routing.allocation.disk.watermark.low：配置low watermark，默认85%。
- cluster.routing.allocation.disk.watermark.high：配置high watermark，默认90%。
- cluster.routing.allocation.disk.watermark.flood_stage：配置flood stage watermark，默认95%。

后面3个配置阈值的配置项除了可以使用百分比以外，也可以使用具体的值，比如配置为low watermark为10gb，表示剩余空闲磁盘低于10gb的时候，就认为到low watermark了。但是需要注意，要么3个配置项都配置百分比，要么都配置具体的值，不允许百分比和具体的值混用。
另外需要注意：如果一个节点配置了多个磁盘，决策时会采用磁盘使用最高的那个。比如一个节点有2个磁盘，一个磁盘使用是84%，一个使用是86%，那也认为该节点进入low watermark了。
最后，如果节点进入flood stage watermark阶段，涉及的索引被设置成read-only以后，如何恢复呢？第一步当然是先通过删数据或增加磁盘/节点等方式让磁盘使用率降到flood stage watermark的阈值以下。然后第二步就是恢复索引状态，取消只读。在7.4.0及以后版本，一旦检测到磁盘使用率低于阈值后，会自动恢复；7.4.0以前的版本，必须手动执行以下命令来取消只读状态：
```
// 恢复单个索引
PUT /索引名称/_settings
{
  "index.blocks.read_only_allow_delete": null
}

// 恢复所有索引
PUT _settings
{
    "index": {
        "blocks": {"read_only_allow_delete": null}
    }
}

// curl命令
curl -XPUT -H "Content-Type: application/json" http://localhost:9200/_all/_settings -d '{"index.blocks.read_only_allow_delete": null}'
```
下面看一下具体的例子。还是前面node-42和node-43组成的集群，每个节点的磁盘总空间是1GB。
> 小技巧：为了方便验证这个功能，Linux用户可以挂载小容量的内存盘来进行操作。这样既免去了没有多个磁盘的烦恼，而且磁盘大小可以设置的比较小，容易操作。比如我测试的2个节点的数据目录就是挂载的1GB的内存盘。

为了方便验证和看的更加清楚，我重新设置几个watermark的阈值，这里不使用百分比，而是用具体的值。
```
PUT _cluster/settings
{
  "transient": {
    "cluster.routing.allocation.disk.watermark.low": "800mb",
    "cluster.routing.allocation.disk.watermark.high": "600mb",
    "cluster.routing.allocation.disk.watermark.flood_stage": "500mb",
    "cluster.info.update.interval": "10s"
  }
}

# 检查下磁盘使用
GET _cat/nodes?v&h=n,dt,du,dup
n        dt    du  dup
node-43 1gb 328kb 0.03
node-42 1gb 328kb 0.03
```
也就是当空闲磁盘低于800mb时进入low watermark；低于600mb时进入high watermark；低于500mb时进入flood stage；每10秒检查一次。接下来写入一些数据，先让磁盘进入low watermark。
```
// 为了查看方便，使用1个 primary shard，1个副本
PUT shard_alloc_test
{
  "settings": {
      "index": {   
          "number_of_shards" : "1",          
          "number_of_replicas": "1"
      }
  }   
}

// 磁盘低于800mb，进入low watermark
GET _cat/nodes?v&h=n,dt,du,dup,da
n        dt      du   dup      da
node-43 1gb 236.1mb 23.06 787.8mb
node-42 1gb 236.1mb 23.06 787.8mb

// 进入low watermark时的日志
[2021-06-01T10:24:59,795][INFO ][o.e.c.r.a.DiskThresholdMonitor] [node-42] low disk watermark [800mb] exceeded on [jgfZk3snRb-uEUToQgo9pw][node-43][/tmp/es-data/nodes/0] free: 799mb[78%], replicas will not be assigned to this node
[2021-06-01T10:24:59,799][INFO ][o.e.c.r.a.DiskThresholdMonitor] [node-42] low disk watermark [800mb] exceeded on [0KbNzeuGR1iZ1I1_3fIDVg][node-42][/tmp/es-data/nodes/0] free: 799mb[78%], replicas will not be assigned to this node
```
可以看到，空闲磁盘小于800MB的时候就进入了low watermark，ES也有对应的日志提示。这个时候副本已经不能分配到这节点上了，我们新建一个索引shard_alloc_test1验证一下。
```
PUT shard_alloc_test1
{
  "settings": {
      "index": {   
          "number_of_shards" : "1",          
          "number_of_replicas": "1"
      }
  }   
}

GET _cat/shards?v
index                               shard prirep state      docs  store ip        node
shard_alloc_test                    0     p      STARTED       0  3.9mb 10.8.4.43 node-43
shard_alloc_test                    0     r      STARTED       0  3.9mb 10.8.4.42 node-42
shard_alloc_test1                   0     p      STARTED       0   208b 10.8.4.43 node-43
shard_alloc_test1                   0     r      UNASSIGNED
```
可以看到，shard_alloc_test1的primary shard分配了，但replica shard没有分配，符合预期。再接着往shard_alloc_test写数据，让进入high watermark。
```
GET _cat/nodes?v&h=n,dt,du,dup,da
n        dt      du   dup      da
node-43 1gb 468.2mb 45.73 555.7mb
node-42 1gb 468.1mb 45.72 555.8mb

[2021-06-01T10:32:59,875][WARN ][o.e.c.r.a.DiskThresholdMonitor] [node-42] high disk watermark [600mb] exceeded on [jgfZk3snRb-uEUToQgo9pw][node-43][/tmp/es-data/nodes/0] free: 555.8mb[54.2%], shards will be relocated away from this node; currently relocating away shards totalling [0] bytes; the node is expected to continue to exceed the high disk watermark when these relocations are complete
[2021-06-01T10:32:59,876][WARN ][o.e.c.r.a.DiskThresholdMonitor] [node-42] high disk watermark [600mb] exceeded on [0KbNzeuGR1iZ1I1_3fIDVg][node-42][/tmp/es-data/nodes/0] free: 555.7mb[54.2%], shards will be relocated away from this node; currently relocating away shards totalling [0] bytes; the node is expected to continue to exceed the high disk watermark when these relocations are complete
```
可以看到，磁盘低于600MB的时候，进入high watermark。这个时候应该不会往该节点分配任何shard了（同时因为只有2个节点，且都引入high watermark了，所以也无法将节点上的shard迁移到其它节点），我们创建新索引shard_alloc_test2验证一下。
```
PUT shard_alloc_test2
{
  "settings": {
      "index": {   
          "number_of_shards" : "1",          
          "number_of_replicas": "1"
      }
  }   
}


GET _cat/shards?v
index                               shard prirep state      docs  store ip        node
shard_alloc_test                    0     p      STARTED       0 17.5mb 10.8.4.43 node-43
shard_alloc_test                    0     r      STARTED       0 17.5mb 10.8.4.42 node-42
shard_alloc_test2                   0     p      UNASSIGNED                     
shard_alloc_test2                   0     r      UNASSIGNED                     
shard_alloc_test1                   0     p      STARTED       0   208b 10.8.4.43 node-43
shard_alloc_test1                   0     r      UNASSIGNED
```
可以看到，主分片、副本分片的shard都没有分配，符合预期。虽然新创建的索引的shard无法分配，但原有的索引还是可以正常写的，我们继续写数据，使磁盘进入flood stage。
```
GET _cat/nodes?v&h=n,dt,du,dup,da
n        dt      du   dup      da
node-43 1gb 535.3mb 52.28 488.6mb
node-42 1gb 535.8mb 52.33 488.1mb

[2021-06-01T10:42:20,024][WARN ][o.e.c.r.a.DiskThresholdMonitor] [node-42] flood stage disk watermark [500mb] exceeded on [jgfZk3snRb-uEUToQgo9pw][node-43][/tmp/es-data/nodes/0] free: 488.6mb[47.7%], all indices on this node will be marked read-only
[2021-06-01T10:42:20,024][WARN ][o.e.c.r.a.DiskThresholdMonitor] [node-42] flood stage disk watermark [500mb] exceeded on [0KbNzeuGR1iZ1I1_3fIDVg][node-42][/tmp/es-data/nodes/0] free: 488.1mb[47.6%], all indices on this node will be marked read-only
```
顺利进入flood stage，索引被设置为read-only。此时，客户端再次写入会收到类似如下错误（这里是JSON格式的日志）：
```
{
  "error" : {
    "root_cause" : [
      {
        "type" : "cluster_block_exception",
        "reason" : "index [shard_alloc_test] blocked by: [TOO_MANY_REQUESTS/12/disk usage exceeded flood-stage watermark, index has read-only-allow-delete block];"
      }
    ],
    "type" : "cluster_block_exception",
    "reason" : "index [shard_alloc_test] blocked by: [TOO_MANY_REQUESTS/12/disk usage exceeded flood-stage watermark, index has read-only-allow-delete block];"
  },
  "status" : 429
}
```
也就是当你的客户端出现“read-only-allow-delete block”错误日志时，表名ES的磁盘空间已经满了。如果是7.4.0版本之前的ES，除了恢复磁盘空间外，还要手动恢复索引的状态，取消只读。
## 总结
Shard是ES中重要的概念，虽然ES默认会处理大部分分配细节，但了解基本原理可以帮助我们设计合理的方案和解决问题，从而提高系统的可靠性。
