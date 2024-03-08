# RocketMQ 如何保证可靠性优先

> 《RocketMQ实战与原理解析》

## 消息重复问题
对分布式消息队列来说，同时做到确保一定投递和不重复投递是很难的， 也就是所谓的“有且仅有 1 次” 在鱼和熊掌不可兼得的情况下， RocketMQ 选择了确保一定投递成功，保证消息不丢失，但有可能造成消息重复。
消息重复一般情况下不会发生，但是如果消息量大，网络有波动，消息重复就是个大概率事件。比如 Producer 有个函数 setRetryTimesWhenSendFailed, 设置在同步方式下自动重试的次数，默认值是 2 ，这样当第一次发送消息时， Broker 端接收到了消息但是没有正确返回发送成功的状态，就造成了消息重复。
解决消息重复有两种方法：第一种方法是保证消费逻辑的幕等性（多次调用和一次调用效果相同）；另一种方法是维护一个巳消费消息的记录，消费前查询这个消息是否被消费过这两种方法都需要使用者自己实现。
## 动态增减机器
一个消息队列集群由 多台机器组成，持续稳定地提供服务，因为业务需求或硬件故障，经常需要增加或减少各个角色的机器，本节介绍如何在不影响服务稳定性的情况下动态地增减机器。
### 动态增减 NameServer
NameServer 是 RocketMQ 集群的协调者，集群的各个组件是通过 NameServer 获取各种属性和地址信息的。主要功能包括两部分：一个是各个 Broker 定期上报自己的状态信息到 NameServer ；另一个是各个客户端 ，包括 Producer 和 Consumer ，以及命令行工具，通过 NameServer 获取最新的状态信。所以，在启动 Broker 、生产者和消费者之前，必须告诉它们 NameServer 的地址，为了提高可靠性，建议启动多个 NameServer。NameServer 占用资源不多，可以和 Broker 部署在同一台机器。有多个 NameServer 后，减少某个 NameServer 不会对其他组件产生影响。
有四种种方式可设置 NameServer 的地址， 下面按优先级由高到低依次介绍： 
1 ）通过代码设置，比如在 Producer 中，通过 Producer.setNamesrvAddr (”name-server1-ip:port;name-server2-ip:port”) 来设 mqadmin 命令行工具中，是通过-n name-server-ip1:port;name-server-ip2:port 参数来设置的 如果自 定义了命令行工具，也可以通过 defaultMQAdminExt.setNamesrvAddr(” nameserver1-ip：port;name-server2-ipport”) 来设置的。
2 ）使用 Java 启动参数设置，对应 option 是 rocketmq.namesrv.addr。
3 ）通过 Linux 环境变量设置，在启动前设置变量： NAMESRV_ADDR。
4 ）通过 HTTP 服务来设置，当上述方法都没有使用，程序会向一个 HTTP 地址发送请求来获取 NameServer 地址，默认的 URL [http://jmenv.tbsite.net:8080/rocketmq/nsaddr](http://jmenv.tbsite.net:8080/rocketmq/nsaddr) （淘宝的测试地址），通过 rocketmq.namesrv.domain 参数来覆盖 jmenv.tbsite.net ；通过 rocketmq.namesrv.domain.subgroup 参数来覆盖 nsaddr 种方式看似繁琐，但它是唯一支持动态增加 NameServer ，无须重启其他组件的方式。使用这种方式后其他组件会每隔分钟请求一次该 URL ，获取最新的 NameServer 地址。
### 动态增减 Broker
由于业务增长,需要对集群进行扩容的时候,可以动态增加 Broker 角色的机器。只增加 Broker 不会对原有的 Topic 产生影响,原来创建好的 Topic 中数据的读写依然在原来的那些 Broker上进行。
集群扩容后,一是可以把新建的 Topic 指定到新的 Broker 机器上,均衡利用资源;另一种方式是通过 updateTopic 命令更改现有的 Topic 配置,在新加的 Broker 上创建新的队列。比如 TestTopic 是现有的一个 Topic ,因为数据量增大需要扩容,新增的一个 Broker机器地址是 192.168.0.1:10911 ,这个时候执行下面的命令: sh ./bin/mqadmin updateTopic -b 192.168.0.1:1091 -t TestTopic -n 192.168.0.100:9876 ,结果是在新增的 Broker机器上,为 TestTopic 新创建了 8 个读写队列
如果因为业务变动或者置换机器需要减少 Broker ,此时该如何操作呢?减少 Broker 要看是否有持续运行的 Producer ,当一个 Topic 只有一个 Master Broker,停掉这个 Broker后,消息的发送肯定会受到影响,需要在停止这个 Broker 前,停止发送消息。
当某个 Topic 有多个 Master Broker,停了其中一个,这时候是否会丢失消息呢? 答案和 Producer 使用的发送消息的方式有关,如果使用同步方式 send(msg) 发送,在 DefaultMQProducer内部有个自动重试逻辑,其中个 Broker停了,会自动向另一个 Broker发消息,不会发生丢消息现象。如果使用异步方式发送send(msg, callback),或者用 sindOne Way方式,会丢失切换过程中的消息。因为在异步和 sindOneWay 这两种发送方式下 Producer.setRetryTimesWhenSendFailed 设置不起作用,发送失败不会重试 DefaultMQProducer 默认每30秒到 Name Server 请求最新的路由消息, Producer 如果获取不到已停止的 Broker下的队列信息,后续就自动不再向这些队列发送消息。
如果 Producer 程序能够暂停,在有一个 Master 和一个 Slave 的情况下也可以顺利切换。可以关闭 Producer后关闭 Master Broker,这个时候所有的读取都会被定向到 Slave机器,消费消息不受影响。把 Master Broker机器置换完后基于原来的数据启动这个 Master Broker,然后再启动 Producer 程序正常发送消息。
用 Linux的 kill pid 命令就可以正确地关闭 Broker, BrokerController下有个 shutdown 函数,这个函数被加到了 Shutdown Hook里,当用 Linux 的 klt 命令时(不能用 kll -9 ), shutdown函数会先被执行。也可以通过 RocketMQ 提供的工具 ( mqshutdown broker ) 来关闭 Broker,它们的原理是一样的。
## 各种故障对消息的影响
我们期望消息队列集群一直可靠稳定地运行,但有时候故障是难免的,本节我们列出可能的故障情况,看看如何处理 : 
1) Broker正常关闭,启动;
2) Broker异常 Crash,然后启动;
3) OS Crash,重启;
4)机器断电,但能马上恢复供电
5)磁盘损坏;
6)CPU、主板、内存等关键设备损坏。
假设现有的 RocketMQ 集群,每个 Topic 都配有多 Master 角色的 Broker 供入,并且每个 Master都至少有一个 Slave 机器(用两台物理机就可以实现上述配置),我们来看看在上述情况下消息的可靠性情况。
第1种情况属于可控的软件问题,内存中的数据不会丢失。如果重启过程中有持续运行的 Consumer , Master 机器出故障后, Consumer 会自动重连到对应的 Slave 机器,不会有消息丢失和偏差。当 Master角色的机器重启以后, Consumer 又会重新连接到 Master 机器 (注意在启动 Master机器的时候, 如果 Consumer 正在从 Slave 消费消息,不要停止 Consumer。假如此时先停止 Consumer 后再启动 Master 机器,然后再启动 Consumer ,这个时候 Consumer 就会去读 Master 机器上已经滞后的 offset值,造成消息大量重复)。
如果第1种情况出现时有持续运行的 Producer,一台 Master出故障后 Producer 只能向 Topic 下其他的 Master 机器发送消息,如果 Producer 采用同步发送方式,不会有消息丢失。
第2、3、4种情况属于软件故障,内存的数据可能丢失,所以刷盘策略不同,造成的影响也不同,如果 Master、 Slave 都配置成 SYNC_FLUSH,可以达到和第1种情况相同的效果。
第5、6种情况属于硬件故障,发生第5、6种情况的故障,原有机器的磁盘数据可能会丢失。如果 Master和 Slave机器间配置成同步复制方式,某一台机器发生 5 或 6 的故障,也可以达到消息不丢失的效果。如果 Master 和 Slave 机器间是异步复制,两次Sync间的消息会丢失。
总的来说,当设置成 : 
1) 多 Master,每个 Master带有 Slave;
2) 主从之间设置成 SYNC_MASTER;
3) Producer 用同步方式写;
4) 刷盘策略设置成 SYNC_FLUSH;
就可以消除单点依赖，即使某台机器出现极端故障也不会丢消息。
## 消息的优先级
有些场景，需要应用程序处理几种类型的消息，不同消息的优先级不同。RocketMQ 是个先进先出的队列，不支持消息级别或者 Topic 级别的优先级。业务中简单的优先级需求，可以通过间接的方式解决，下面列举三种优先级相关需求的具体处理方法。
第一种是比较简单的情况，如果当前 Topic 里有多种相似类型的消息，比如类型 AA、AB、AC ，当 AB、AC 的消息量很大，但是处理速度比较慢的时候， 队列里会有很多 AB、AC 类型的消息在等候处理，这个时候如果有少量 AA 型的消息加人，就会排在 AB、AC 类型消息后面，需要等候很长时间才能被处理。如果业务需要 AA 类型的消息被及时处理，可以把这三种相似类型的消息 分拆到两个 Topic 里，比如 AA 类型的消息在一个单独的 Topic, AB、AC 类型 的消息在另外一个 Topic 把消息分到两个 Topic 中以后，应用程序创建两个 Consumer ，分别订阅不同的 Topic ，这样消息 AA 在单独的 Topic 里，不会因 AB、AC 类型的消息太多而被长时间延时处理。
第二种情况和第一种情况类似，但是不用创建大量的 Topic 。举个实际应用场景：一个订单处理系统，接收从 100 家快递门店过来的请求，把这些请求 通过 Producer 写人 RocketMQ ；订单处理程序通过 Consumer 从队列里读取消息并处理，每天最多处理 1 万单。如果这 100 个快递门店中某几个门店订单量大增，比如门店一接了个大客户，一个上午就发出 1 万单消息请求，这样其他 99 家门店可能被迫等待门店一的 1 万单处理完，也就是两天后订单才能被处理，显然很不公平。这时可以创建一个 Topic ,设置 Topic 的 MessageQueue 数量超过100个, Producer根据订单的门店号,把每个门店的订单写人一个 MessageQueue。DefaultMQPushConsumer 默认是采用循环的方式逐个读取一个 Topic 的所有 MessageQueue ,这样如果某家门店订单量大增,这家门店对应的 MessageQueue 消息数增多,等待时间增长,但不会造成其他家门店等待时间增长。DefaultMQPushConsumer 默认的 pullBatchSize是32,也就是每次从某个 MessageQueue 读取消息的时候,最多可以读32个。在上面的场景中,为了更加公平,可以把 pullBatchSize设置成 1。
第三种情况是强优先级需求,上两种情况对消息的“优先级”要求不高,更像一个保证公平处理的机制,避免某类消息的增多阻塞其他类型的消息。现在有一个应用程序同时处理 TypeA、TypeB、TypeC 三类消息。 TypeA 处于第一优先级,要确保只要有 TypeA 消息,必须优先处理; TypeB 处于第二优先级; TypeC 处于第三优先级。对这种要求,或者逻辑更复杂的要求,就要用户自己编码实现优先级控制,如果上述的三类消息在一个 Topic里,可以使用 PullConsumer ,自主控制 MessageQueue 的遍历,以及消息的读取;如果上述三类消息在三个 Topic 下,需要启动三个 Consumer ,实现逻辑控制三个 Consumer 的消费。
