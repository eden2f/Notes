# Redis BigKey的影响及相关工具

> [Redis scan命令详解](http://doc.redisfans.com/key/scan.html)
[阿里云redis大key搜索工具](https://developer.aliyun.com/article/117042)
[Redis—大key问题讨论及解决方案](https://www.pianshen.com/article/33531420666/)

## 什么是 BigKey
所谓的bigkey就是存储本身的key值空间太大，或者hash，list，set等存储中value值过多。
(实际上, 这个定义比较抽象, 不同的业务对其有不同的理解.)
主要包括：

- 单个简单的key存储的value很大
- hash， set，zset，list 中存储过多的元素
- 一个集群存储了上亿的key

bigkey会带来一些问题，如：

- 读写bigkey会导致超时严重，甚至阻塞服务。
- 大key相关的删除或者自动过期时，会出现qps突降或者突升的情况，极端情况下，会造成主从复制异常，Redis服务阻塞无法响应请求。bigkey的体积与删除耗时可参考下表：
| key类型 | field数量 | 耗时 |
| --- | --- | --- |
| Hash | 100万 | 1000ms |
| List | 100万 | 1000ms |
| Set | 100万 | 1000ms |
| ZSet | 100万 | 1000ms |

Redis 是处理业务请求是单线程作业的，操作 BigKey 比较耗时，那么阻塞 Redis 的可能性增大。每次获取 BigKey的网络流量较大，假设一个 BigKey 为 1MB,每秒访问量为 1000，那么每秒产生 1000MB 的流量，对于普通千兆网卡，按照字节算 128M/S 的服务器来说可能扛不住。而且一般服务器采用单机多实例方式来部署，所以还可能对其他实例造成影响。
## 如何扫描 BigKey
Redis是处理业务请求是单线程处理的，肯定不能用`keys`命令来筛选，因为keys命令会一次性进行全盘搜索，会造成redis的阻塞，从而会影响正常业务的命令执行。
对于 Redis 主从版本可以通过scan命令进行扫描，对于集群版本提供了ISCAN命令进行扫描，命令规则如下, 其中节点个数node可以通过info命令来获取到。
```shell
SCAN cursor [MATCH pattern] [COUNT count]
```
```shell
ISCAN idx cursor [MATCH pattern] [COUNT count]（idx为节点的id，从0开始，16到64gb的集群实例为8个节点故idx为0到7，128g 256gb的为16个节点）
```
SCAN 命令及其相关的 SSCAN 命令、 HSCAN 命令和 ZSCAN 命令都用于增量地迭代（incrementally iterate）一集元素（a collection of elements）：
SCAN 命令用于迭代当前数据库中的数据库键。
SSCAN 命令用于迭代集合键中的元素。
HSCAN 命令用于迭代哈希键中的键值对。
ZSCAN 命令用于迭代有序集合中的元素（包括元素成员和元素分值）。
以上列出的四个命令都支持增量式迭代， 它们每次执行都只会返回少量元素， 所以这些命令可以用于生产环境， 而不会出现像 KEYS 命令、 SMEMBERS 命令带来的问题 —— 当 KEYS 命令被用于处理一个大的数据库时， 又或者 SMEMBERS 命令被用于处理一个大的集合键时， 它们可能会阻塞服务器达数秒之久。
不过， 增量式迭代命令也不是没有缺点的： 举个例子， 使用 SMEMBERS 命令可以返回集合键当前包含的所有元素， 但是对于 SCAN 这类增量式迭代命令来说， 因为在对键进行增量式迭代的过程中， 键可能会被修改， 所以增量式迭代命令只能对被返回的元素提供有限的保证 （offer limited guarantees about the returned elements）。
## 阿里云redis大key搜索工具
### 初始化环境
#### 安装python客户端
下载python客户端
wget “https://pypi.python.org/packages/68/44/5efe9e98ad83ef5b742ce62a15bea609ed5a0d1caf35b79257ddb324031a/redis-2.10.5.tar.gz#md5=3b26c2b9703b4b56b30a1ad508e31083”
#### 解压安装
tar -xvf redis-2.10.5.tar.gz
cd redis-2.10.5
sudo python setup.py install
### 扫描脚本
```python
import sys
import redis

def check_big_key(r, k):
  bigKey = False
  length = 0 
  try:
    type = r.type(k)
    if type == "string":
      length = r.strlen(k)
    elif type == "hash":
      length = r.hlen(k)
    elif type == "list":
      length = r.llen(k)
    elif type == "set":
      length = r.scard(k)
    elif type == "zset":
      length = r.zcard(k)
  except:
    return
  if length > 10240:
    bigKey = True
  if bigKey :
    print db,k,type,length

def find_big_key_normal(db_host, db_port, db_password, db_num):
  r = redis.StrictRedis(host=db_host, port=db_port, password=db_password, db=db_num)
  for k in r.scan_iter(count=1000):
    check_big_key(r, k)

def find_big_key_sharding(db_host, db_port, db_password, db_num, nodecount):
  r = redis.StrictRedis(host=db_host, port=db_port, password=db_password, db=db_num)
  cursor = 0
  for node in range(0, nodecount) :
    while True:
      iscan = r.execute_command("iscan",str(node), str(cursor), "count", "1000")
      for k in iscan[1]:
        check_big_key(r, k)
      cursor = iscan[0]
      print cursor, db, node, len(iscan[1])
      if cursor == "0":
        break;
  
if \__name__\ == '__main__':
  if len(sys.argv) != 4:
     print 'Usage: python ', sys.argv[0], ' host port password '
     exit(1)
  db_host = sys.argv[1]
  db_port = sys.argv[2]
  db_password = sys.argv[3]
  r = redis.StrictRedis(host=db_host, port=int(db_port), password=db_password)
  nodecount = r.info()['nodecount']
  keyspace_info = r.info("keyspace")
  for db in keyspace_info:
    print 'check ', db, ' ', keyspace_info[db]
    if nodecount > 1:
      find_big_key_sharding(db_host, db_port, db_password, db.replace("db",""), nodecount)
    else:
      find_big_key_normal(db_host, db_port, db_password, db.replace("db", ""))
```
可以通过python find_bigkey host 6379 password来执行，支持阿里云Redis的主从版本和集群版本的大key查找，默认大key的阈值为10240，也就是对于string类型的value大于10240的认为是大key，对于list的话如果list长度大于10240认为是大key，对于hash的话如果field的数目大于10240认为是大key。另外默认该脚本每次搜索1000个key，对业务的影响比较低，不过最好在业务低峰期进行操作，避免scan命令对业务的影响。
