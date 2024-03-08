# RocketMQ 消息队列的核心 Broker

> 《RocketMQ实战与原理解析》

## 前言
Broker 是 RocketMQ 的核心, 大部分“重量级”工作都是由 Broker 完成的, 包括接收 Producer 发过来的消息、处理 Consumer 的消费消息请求、消息的持久化存储、以及服务端过滤功能等。
## 消息存储和发送
分布式队列因为有高可靠性的要求,所以数据要通过磁盘进行持久化存储。用磁盘存储消息,速度会不会很慢呢?能满足实时性和高吞吐量的要求吗?实际上,磁盘有时候会比你想象的快很多,有时候也会比你想象的慢很多关键在如何使用,使用得当,磁盘的速度完全可以匹配上网络的数据传输速度。目前的高性能磁盘,顺序写速度可以达到600MB/s,超过了一般网卡的传输速度,这是磁盘比想象的快的地方。但是磁盘随机写的速度只有大概100KB/s,和顺序写的性能相差6000倍!因为有如此巨大的速度差别,好的消息队列系统会比普通的消息队列系统速度快多个数量级。
举个例子, Linux操作系统分为“用户态”和“内核态”,文件操作、网络操作需要涉及这两种形态的切换，免不了进行数据复制，一台服务器把本机磁 盘文件的内容发送到客户端 般分为两个步骤：

1.  read(file, tmp buf, len); ，读取本地文件内容； 
2.  write(socket, tmp_buf, len); ，将读取的内容通过网络发送出去 。
tmp_buf 是预先申请的内存，这两个看似简单的操作，实际进行了4次数据复制，分别是：从磁盘复制数据到内核态内存，从内核态内存复制到用户态内存（完成了  read(file, tmp_b len ；然后从用户态内存复制到网络驱动的内核态内存，最后是从网络驱动的内核态内存复制到网卡中进行传输（完成 write(socket, tmp_buf, len)） 。
通过使用 mmap 的方式，可以省去向用户态的内存复制，提高速度种机制在 Java 中是通过 MappedByteBuffer 实现的，具体可以参考 Java 7 的文 档：https://docs.oracle.com/javase/7/docs/api/java/nio/MappedByteBuffer.html。RocketMQ 充分利用了上述特性，也就是所谓的“零拷贝”技术，提高消息存盘和网络发送的速度。 
## 消息存储结构
RocketMQ 的具体消息存储结构是怎样的呢？如何尽量保证顺序写的呢？ 先来看看整体的架构图 :
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240308235940.png#id=Dvenh&originHeight=388&originWidth=582&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
RocketMQ 消息的存储是由 ConsumeQueue CommitLog 配合完成的， 消息真正的物理存储文件是 CommitLog, ConsumeQueue 是消息的逻辑队列，类似数据库的索引文件，存储的是指向物理存储的地址每个 Topic 的每个 Message Queue 都有一个对应的 ConsumeQueue 文件的文件地址在如下位置。
```
${$storeRoot}\consumequeue\${topicName}\$ { queueld}\$ {fileName
```
CommitLog 以物理文件的方式存放，每台 Broker 上的 CommitLog 被本机器所有 ConsumeQueue 共享。CommitLog 中，一个消息的存储长度是不固定的， RocketMQ 采取一些机制，尽 CommitLog 中顺序写 ，但是随机读 ConsumeQueue 被写到磁盘里作持久存储。
```
${user.home}\store\${commitlog}\${fileName}
```
存储机制这样设计有以下几个好处： 
1 ) CommitLog 顺序写 ，可以大大提高写入效率
2 ) 虽然是随机读，但是利用操作系统的 pagecache 机制，可以批量地从磁盘读取，作为 cache 存到内存中，加速后续的读取速度。
3 ) 为了保证完全的顺序写,需要 ConsumeQueue 这个中间结构,因为
consumeQueue 里只存偏移量信息,所以尺寸是有限的,在实际情况中,大部
分的 ConsumeQueue 能够被全部读入内存,所以这个中间结构的操作速度很快
可以认为是内存读取的速度。此外为了保证 Commitlog 和 ConsumeQueue 的一致性, Commitlog 里存储了 ConsumeQueues、MessageKey、Tag 等所有信息
即使 ConsumeQueue 丢失,也可以通过 commitLog 完全恢复出来。
下图为 RocketMQ 的 Broker 机器磁盘上的文件存储结构 :
![](https://gitee.com/eden2f/ImageHosting/raw/master/imgs/20210418231817.png#id=YvFd5&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240308235452.png#id=r5Vw7&originHeight=293&originWidth=397&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
## 高可用性机制
RocketMQ 分布式集群是通过 Master 和 Slave 的配合达到高可用性的,首先说一下 Master 和 Slave 的区别 : 在 Broker 的配置文件中,参数 brokerId 的值为 0 表明这个 Broker 是 Master ，大于 0 表明这个 Broker 是 Slave ，同时 brokerRole 参数也会说明这个 Broker 是 Master 还是 Slave。Master 角色的 Broker 支持读和写， Slave 角色的 Broker 仅支持读，也就是 Producer 只能和 Master 角色的 Broker 连接写人消息； Consumer 可以连接 Master 角色的 Broker ，也可以连接 Slave 角色的 Broker 来读取消息。
在 Consumer 的配置文件中，并不需要设置是从 Master 读还是从 Slave 读，当 Master 不可用或者繁忙的时候， Consumer 会被自动切换到从 Slave 读。有了自动切换 Consumer 这种机制，当一个 Master 角色的机器出现故障后， Consumer 仍然可以从 Slave 读取消息，不影响 Consumer 。这就达到了消费端的高可用性。
如何达到发送端的高可用性呢？在创建 Topic 的时候，把 Topic 的多个 Message Queue 创建在多个 Broker 组上（相同 Broker 名称，不同 brokerId 机器组成一个 Broker 组），这样当 Broker 组的 Master 不可用后，其他组 Master 仍然可用， Producer 仍然可以发送消息。 RocketMQ 目前还不支持把 Slave 自动转成 Master ，如果机器资源不足，需要把 Slave 转成 Master ，则要手动停止 Slave 角色的 Broker ，更改配置文件，用新的配置文件启动 Broker。
## 同步刷盘和异步刷盘
RocketMQ 的消息是存储到磁盘上的，这样既能保证断电后恢复，又可以让存储的消息超出内存的限制。RocketMQ 为了提高性能，会尽可能地保证磁盘的顺序写。消息在通过 Producer 写人 RocketMQ 的时候，有两种写磁盘方式，下面逐一介绍。
异步刷盘方式：在返回写成功状态时 ，消息可能只是被写人了内存的 PAGECACHE ，写操作的返回快，吞吐量大 ；当内存里的消息 积累到 一定程度时 ，统一触发写磁盘动作，快速写人。
同步刷盘方式：在返回写成功状态时，消息已经被写人磁盘。具体流程是，消息写入内存的 PAGECACHE 后，立刻通知刷盘线程刷盘，然后等待刷盘完成，刷盘线程执行完成后唤醒等待的线程，返回消息写成功的状态。
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240308235600.png#id=jJAne&originHeight=353&originWidth=435&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
同步刷盘还是异步刷盘，是通过 Broker 配置文件里的 flushDiskType 参数设置的，这个参数被配置成 SYNC_FLUSH、ASYNC_FLUSH 中的一个。
## 同步复制和异步复制
如果一个 Broker 组有 Master Slave, 消息需要从 Master 复制到 Slava 上，有同步和异步两种复制方式。同步复制方式是 Master 和 Slave 均写成功后才反馈给客户端写成功状态；异步复制方式是只要 Master 写成功即可反馈给客户端写成功状态。
这两种复制方式各有优劣,在异步复制方式下,系统拥有较低的延迟和较高的吞吐量,但是如果 Master 出了故障,有些数据因为没有被写入 Slave,有可能会丢失;在同步复制方式下,如果 Master 出故障, Slave上有全部的备份
数据,容易恢复,但是同步复制会增大数据写入延迟,降低系统吞吐量。同步复制和异步复制是通过 Broker 配置文件里的 brokerRole 参数进行设置的,这个参数可以被设置成 ASYNC_MASTER、 SYNC_MASTER、 SLAVE 三个值中的一个。
实际应用中要结合业务场景,合理设置刷盘方式和主从复制方式,尤其是 SYNC_FLUSH 方式,由于频繁地触发磁盘写动作,会明显降低性能。通常情况下,应该把 Master 和 Slave 配置成 ASYNC_FLUSH 的刷盘方式,主从之间配置成 SYNC_MASTER 的复制方式,这样即使有一台机器出故障,仍然能保证数据不丢,是个不错的选择。
