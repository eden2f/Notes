# Elasticsearch 从重复文档问题到Elasticsearch请求过程分析

> [Elasticsearch Guide](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)
[Elasticsearch High Level Rest Client 发起请求的过程分析](https://www.cnblogs.com/hapjin/p/10116073.html)

## 前言
由于我在同步 MySQL 数据到 Elasticsearch 过程中，并没有自定义 Elasticsearch 的文档的主键，用了 Elasticsearch 默认的主键生成策略。（Elasticsearch mapping 结构在下文展示）
问题来了，在将 MySQL 存量数据迁移至 Elasticsearch 时，发现出现了userId重复的文档，他们的 ”_id” 并不一致。（如何查找 Elasticsearch 中重复的文档可以参考这篇文章《Elasticsearch 重复文档的查找与消除》）
我马上想到 Elasticsearch 支持设置数据分片，而且我连接 Elasticsearch 环境是由三个 Elasticsearch 实例组成集群。所以，利用 redis 简单做个分布式锁，伪代码如下所示，然后又重新做了一次数据迁移。
```java
public void syncByUserId(Long userId) {
    String redisKey = REDIS_KEY_PREFIX + userId;
    try {
        // redisUtil.加锁(redisKey);
    } catch (EsSyncConcurrentLockException e) {
        // 抢锁失败
        return;
    }
    try {
        // 查询 MySQL 内容
        User po = selectMySqlInfoByUserId(qcCode);
        // 同步到 Elasticsearch 用 RestHighLevelClient 实现 
        // 第一步：先根据userId查询Es
        // 第二步: 如果userId不存在，新增文档；如果userId已存在，更新文档；
        insertOrUpdate(po);
    } finally {
        // redisUtil.释放锁(redisKey);
    }
}
```
又是一轮数据迁移……发现文档重复的问题依然存在，我愈发困惑了，难道是 Elastic Client 的问题（也就是 RestHighLevelClient ），我决心看一下底层实现，也就发现了这一篇文章 [Elasticsearch High Level Rest Client 发起请求的过程分析](https://www.cnblogs.com/hapjin/p/10116073.html) 写得挺不错的，看了下源码，RestClient 会根据 nodes 做负载均衡。但是，我本地测试时只配置了一个节点，难道 RestClient  根据可用节点拉取该集群下所有节点？找了很久没有找到相关的代码实现，也可能自己没找到 “对” 的地方？为了排除这个可能，我直接将集群节点从三个改为一个，现在就只有一个节点。
再来一轮数据迁移……问题依旧扎心，但好的是，根据这一次结果我知道，我的问题，跟集群数量无关、也不是同一 userId 并发 insert 导致。根据自己的经验，跑到 Elasticsearch 官方文档找相关问题，才发现了问题根源。
Elasticsearch insert 后并不会马上刷新分片数据，默认是一秒，支持配置。
[官方原文 : Elasticsearch Guide [7.13] » REST APIs » Document APIs » ?refresh](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-refresh.html)
## 实时配置参数 refresh
对于 Index, Update, Delete, 和 Bulk 相关API支持设置`refresh`，以控制当申请所做的更改可见搜索。
`refresh` 参数支持以下传值：
空字符串或 `true`
操作发生后立即刷新相关的主分片和副本分片（而不是整个索引），以便更新的文档立即出现在搜索结果中。只有在仔细考虑和验证它不会导致性能不佳后，才应该这样做，无论是从索引还是搜索的角度来看。
`wait_for`
在回复之前等待通过刷新使请求所做的更改可见。这不会强制立即刷新，而是等待刷新发生。Elasticsearch 会自动刷新每更改一次的分片，`index.refresh_interval`默认为一秒。该设置是 [动态的](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules.html#dynamic-index-settings)。在任何支持它的 API 上调用[Refresh](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-refresh.html) API 或设置`refresh`为`true`也会导致刷新，进而导致已经运行的请求`refresh=wait_for` 返回。
`false` （默认）
不执行与刷新相关的操作。此请求所做的更改将在请求返回后的某个时间点可见。
## 示例 Elasticsearch mapping 结构
index ： user_detail (存储用户相关信息)
```json
{
    "properties": {
        "userId": {
            "type": "long"
        },
        "name": {
            "type": "keyword"
        },
        "age": {
            "type": "integer"
        },
        "email": {
            "type": "keyword"
        },
        "headPortrait": {
            "type": "text"
        },
        "imgs": {
            "type": "text"
        },
        "userOperationLogs": {
            "properties": {
                "id": {
                    "type": "long"
                },
                "userId": {
                    "type": "long"
                },
                "desc": {
                    "type": "text"
                }
            }
        }
    }
}
```
## RestHighLevelClient 发起请求的过程分析
本文讨论的是JAVA High Level Rest Client向ElasticSearch6.3.2发送请求(index操作、update、delete……)的一个详细过程的理解，主要涉及到Rest Client如何选择哪一台Elasticsearch服务器发起请求。
maven依赖如下：
```html
<dependency>
    <groupId>org.elasticsearch.client</groupId>
    <artifactId>elasticsearch-rest-high-level-client</artifactId>
    <version>6.3.2</version>
</dependency>
```
High Level Rest Client 为这些请求提供了两套接口：同步和异步，异步接口以Async结尾。以update请求为例，如下：
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240317203010.png#id=tTSx5&originHeight=70&originWidth=840&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
官方也提供了详细的示例来演示如何使用这些API：[java-rest-high](https://www.elastic.co/guide/en/elasticsearch/client/java-rest/current/java-rest-high.html#java-rest-high)，在使用之前需要先[初始化](https://www.elastic.co/guide/en/elasticsearch/client/java-rest/current/java-rest-high-getting-started-initialization.html)一个RestHighLevelClient 然后就可以参考API文档开发了。RestHighLevelClient 底层封装的是一个http连接池，当需要执行 update、index、delete操作时，直接从连接池中取出一个连接，然后发送http请求到ElasticSearch服务端，服务端基于Netty接收请求。
```html
The high-level client will internally create the low-level client used to perform requests based on the provided builder. That low-level client maintains a pool of connections
```
本文的主要内容是探究一下 index/update/delete请求是如何一步步构造，并发送到ElasticSearch服务端的，并重点探讨选择向哪个ElasticSearch服务器发送请求的 round robin 算法
以update请求为例：[构造了update请求](https://www.elastic.co/guide/en/elasticsearch/client/java-rest/current/java-rest-high-document-update.html)后：执行`esClient.update(updateRequest);`发起请求：
```java
updateRequest.doc(XContentFactory.jsonBuilder().startObject().field(fieldName, val).endObject());
            UpdateResponse response = esClient.update(updateRequest);
```
最终会执行到`performRequest()`，index、delete请求最终也是执行到这个方法：
```java
    /**
     * Sends a request to the Elasticsearch cluster that the client points to. Blocks until the request is completed and returns
     * its response or fails by throwing an exception. Selects a host out of the provided ones in a round-robin fashion. Failing hosts
     * are marked dead and retried after a certain amount of time (minimum 1 minute, maximum 30 minutes), depending on how many times
     * they previously failed (the more failures, the later they will be retried). In case of failures all of the alive nodes (or dead
     * nodes that deserve a retry) are retried until one responds or none of them does, in which case an {@link IOException} will be thrown.
     *
     *
     */
    public Response performRequest(String method, String endpoint, Map<String, String> params,
                                   HttpEntity entity, HttpAsyncResponseConsumerFactory httpAsyncResponseConsumerFactory,
                                   Header... headers) throws IOException {
        SyncResponseListener listener = new SyncResponseListener(maxRetryTimeoutMillis);
        performRequestAsyncNoCatch(method, endpoint, params, entity, httpAsyncResponseConsumerFactory,
            listener, headers);
        return listener.get();
    }
```
看这个方法的注释，向Elasticsearch cluster发送请求，并等待响应。等待响应就是通过创建一个`SyncResponseListener`，然后执行`performRequestAsyncNoCatch`先异步把HTTP请求发送出去，然后SyncResponseListener等待获取请求的响应结果，即：`listener.get();`阻塞等待直到拿到HTTP请求的响应结果。
`performRequestAsyncNoCatch()`里面调用的内容如下：
```java
client.execute(requestProducer, asyncResponseConsumer, context, new FutureCallback<HttpResponse>() {
            @Override
            public void completed(HttpResponse httpResponse) {
```
也就是CloseableHttpAsyncClient的execute()方法向ElasticSearch服务端发起了HTTP请求。(rest-high-level client封装的底层http连接池)
以上就是：ElasticSearch JAVA High Level 同步方法的具体执行过程。总结起来就二句：`performRequestAsyncNoCatch`异步发送请求，`SyncResponseListener`阻塞获取响应结果。异步方法的执行方式也是类似的。
在[这篇文章](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-node.html)中提到，ElasticSearch集群中每个节点默认都是Coordinator 节点，可以接收Client的请求。因为在创建ElasticSearch JAVA High Level 时，一般会配置多个IP地址，如下就配置了三台：
```
//	    es中默认 每个节点都是 coordinating node
            String[] nodes = clusterNode.split(",");
            HttpHost host_0 = new HttpHost(nodes[0].split(":")[0], Integer.parseInt(nodes[0].split(":")[1]), "http");
            HttpHost host_1 = new HttpHost(nodes[1].split(":")[0], Integer.parseInt(nodes[1].split(":")[1]), "http");
            HttpHost host_2 = new HttpHost(nodes[2].split(":")[0], Integer.parseInt(nodes[2].split(":")[1]), "http");
            restHighLevelClient = new RestHighLevelClient(RestClient.builder(host_0, host_1, host_2));
```
那么，Client在发起HTTP请求时，到底是请求到了哪台ElasticSearch服务器上呢？这就是本文想要讨论的问题。
而发送请求主要由RestClient实现，看看这个类的源码注释，里面就提到了sending a request, a host gets selected out of the provided ones in a round-robin fashion. 
```java
/**
 * Client that connects to an Elasticsearch cluster through HTTP.
 * The hosts that are part of the cluster need to be provided at creation time, but can also be replaced later
 * The method {@link #performRequest(String, String, Map, HttpEntity, Header...)} allows to send a request to the cluster. When
 * sending a request, a host gets selected out of the provided ones in a round-robin fashion. Failing hosts are marked dead and
 * retried after a certain amount of time (minimum 1 minute, maximum 30 minutes), depending on how many times they previously
 * failed (the more failures, the later they will be retried). In case of failures all of the alive nodes (or dead nodes that
 * deserve a retry) are retried until one responds or none of them does, in which case an {@link IOException} will be thrown.
 * <p>
 * Requests can be either synchronous or asynchronous. The asynchronous variants all end with {@code Async}.
 * <p>
 */
public class RestClient implements Closeable {
    
    //一些代码
    
    
        /**
     * {@code HostTuple} enables the {@linkplain HttpHost}s and {@linkplain AuthCache} to be set together in a thread
     * safe, volatile way.
     */
    private static class HostTuple<T> {
        final T hosts;
        final AuthCache authCache;

        HostTuple(final T hosts, final AuthCache authCache) {
            this.hosts = hosts;
            this.authCache = authCache;
        }
    }
}
```
HostTuple是RestClient是静态内部类，封装在配置文件中配置的ElasticSearch集群中各台机器的IP地址和端口。
因此，对于Client而言，存在2个问题：

1. 怎样选一台“可靠的”机器，然后放心地把我的请求交给它？
2. 如果Client端的请求量非常大，不能老是把请求都往ElasticSearch某一台服务器发，应该要考虑一下负载均衡。

其实具体的算法实现细节我也没有深入去研究理解，不过把这两个问题抽象出来，其实在很多场景中都能碰到。
> 客户端想要连接服务端，服务器端提供了很多主机可供选择，我应该需要考虑哪些因素，选一台合适的主机连接？

在`performRequestAsync`方法的参数中，会调用RestClient类的`netxtHost()`：方法，选择合适的ElasticSearch服务器IP进行连接。
```java
void performRequestAsyncNoCatch(String method, String endpoint, Map<String, String> params,
                                    HttpEntity entity, HttpAsyncResponseConsumerFactory httpAsyncResponseConsumerFactory,
                                    ResponseListener responseListener, Header... headers) {
    
    //省略其他无关代码
        performRequestAsync(startTime, nextHost(), request, ignoreErrorCodes, httpAsyncResponseConsumerFactory,
                failureTrackingResponseListener);
}
 /**
     * Returns an {@link Iterable} of hosts to be used for a request call.
     * Ideally, the first host is retrieved from the iterable and used successfully for the request.
     * Otherwise, after each failure the next host has to be retrieved from the iterator so that the request can be retried until
     * there are no more hosts available to retry against. The maximum total of attempts is equal to the number of hosts in the iterable.
     * The iterator returned will never be empty. In case there are no healthy hosts available, or dead ones to be be retried,
     * one dead host gets returned so that it can be retried.
     */
    private HostTuple<Iterator<HttpHost>> nextHost() {
```
nextHost()方法的大致逻辑如下：
```java
do{
    //先从HostTuple中拿到ElasticSearch集群配置的主机信息
    //....
    
    if (filteredHosts.isEmpty()) {
        //last resort: if there are no good hosts to use, return a single dead one, the one that's closest to being retried
        //所有的主机都不可用，那就死马当活马医
        HttpHost deadHost = sortedHosts.get(0).getKey();
        nextHosts = Collections.singleton(deadHost);
    }else{
        List<HttpHost> rotatedHosts = new ArrayList<>(filteredHosts);
        //rotate()方法选取最适合连接的主机
                Collections.rotate(rotatedHosts, rotatedHosts.size() - lastHostIndex.getAndIncrement());
                nextHosts = rotatedHosts;
    }
    
}while(nextHosts.isEmpty())
```
选择ElasticSearch主机连接主要是由`rotate()`实现的。该方法里面又有2种实现，具体代码就不贴了，看注释：
```java
    /**
     * Rotates the elements in the specified list by the specified distance.
     * After calling this method, the element at index <tt>i</tt> will be
     * the element previously at index <tt>(i - distance)</tt> mod
     * <tt>list.size()</tt>, for all values of <tt>i</tt> between <tt>0</tt>
     * and <tt>list.size()-1</tt>, inclusive.  (This method has no effect on
     * the size of the list.)
     *
     * <p>For example, suppose <tt>list</tt> comprises<tt> [t, a, n, k, s]</tt>.
     * After invoking <tt>Collections.rotate(list, 1)</tt> (or
     * <tt>Collections.rotate(list, -4)</tt>), <tt>list</tt> will comprise
     * <tt>[s, t, a, n, k]</tt>.
     *
     * <p>Note that this method can usefully be applied to sublists to
     * move one or more elements within a list while preserving the
     * order of the remaining elements.  For example, the following idiom
     * moves the element at index <tt>j</tt> forward to position
     * <tt>k</tt> (which must be greater than or equal to <tt>j</tt>):
     * 
<pre>
     *     Collections.rotate(list.subList(j, k+1), -1);
     * </pre>
     * To make this concrete, suppose <tt>list</tt> comprises
     * <tt>[a, b, c, d, e]</tt>.  To move the element at index <tt>1</tt>
     * (<tt>b</tt>) forward two positions, perform the following invocation:
     * 
<pre>
     *     Collections.rotate(l.subList(1, 4), -1);
     * </pre>
     * The resulting list is <tt>[a, c, d, b, e]</tt>.
     *
     * <p>To move more than one element forward, increase the absolute value
     * of the rotation distance.  To move elements backward, use a positive
     * shift distance.
     *
     * <p>If the specified list is small or implements the {@link
     * RandomAccess} interface, this implementation exchanges the first
     * element into the location it should go, and then repeatedly exchanges
     * the displaced element into the location it should go until a displaced
     * element is swapped into the first element.  If necessary, the process
     * is repeated on the second and successive elements, until the rotation
     * is complete.  If the specified list is large and doesn't implement the
     * <tt>RandomAccess</tt> interface, this implementation breaks the
     * list into two sublist views around index <tt>-distance mod size</tt>.
     * Then the {@link #reverse(List)} method is invoked on each sublist view,
     * and finally it is invoked on the entire list.  For a more complete
     * description of both algorithms, see Section 2.3 of Jon Bentley's
     * <i>Programming Pearls</i> (Addison-Wesley, 1986).
     *
     */
    public static void rotate(List<?> list, int distance) {
        if (list instanceof RandomAccess || list.size() < ROTATE_THRESHOLD)
            rotate1(list, distance);
        else
            rotate2(list, distance);
    }
```
