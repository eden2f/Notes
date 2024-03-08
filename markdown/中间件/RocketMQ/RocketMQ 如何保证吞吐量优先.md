# RocketMQ 如何保证吞吐量优先

> 《RocketMQ实战与原理解析》

## 前言
本章介绍在大流量场景下,提高 RocketMQ 集群吞吐量的一些方法,有些方法当服务器出异常时会增大丢消息的概率,用户需要根据业务需求酌情使用。
## 在 Broker端进行消息过滤
在 Broker 端进行消息过滤,可以减少无效消息发送到 Consumer ,少占用网络带宽从而提高吞吐量。 Broker 端有三种方式进行消息过滤。
### 消息的Tag和Key
对一个应用来说,尽可能只用一个 Topic ,不同的消息子类型用 Tag 来标识(每条消息只能有一个 Tag ),服务器端基于 Tag 进行过滤,并不需要读取消息体的内容,所以效率很高。发送消息设置了 Tag 以后,消费方在订阅消息时,才可以利用 Tag 在 Broker 端做消息过滤。
其次是消息的Key。对发送的消息设置好 Key ,以后可以根据这个 Key 来查找消息。所以这个 Key 一般用消息在业务层面的唯一标识码来表示,这样后续查询消息异常，消息丢失等都很方便 Broker 会创建专门的索引文件，来存 Key 到消息的映射，由于是哈希索引，应尽 Key ，避免潜在的哈希冲突。
Tag 和 Key 的主要差别是使用场景不同， Tag 用在 Consumer 的代码中，用来进行服务端消息过滤， Key 主要用于通过命令行查询消息。
### 通过 Tag 进行过滤
用 Tag 方式进行过滤的方法是传人感兴趣的 Tag 标签，Tag 标签是一个普通字符串，是在创建 Message 的时候添加的， Message 只能有一个 Tag 使用 Tag 方式过滤非常高效， Broker 端可以在 ConsumeQueue 中做这种过洁 只从 CommitLog 里读取过滤后被命中的消息 ConsumerQueue 的存储 格式，如图所示
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240309001150.png#id=ovl8A&originHeight=102&originWidth=510&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
Consume Queue 部分存储的是 Tag 对应的 hashcode ，是一个定长的 字符串，通过 Tag 过滤的过程就是对比定长的 hashcode。经过 hashcode 对比, 符合要求的消息被从 CommitLog 读取出来，不用担心 Hash 冲突问题，消息在被消费前，会对比完整的 Message Tag 字符串，消除 Hash 冲突造成的误读。
### 用 SQL 表达式的方式进行过滤
使用 Tag 方式过滤虽然高效，但是支持的逻辑比较简单，在构造 Message 的时候，还可以通过 putUserProperty 函数来增加多个自定义的属性，基于这些属性可以做复杂的过逻辑。
```java
Message msg =new Message("TopicTest", tag, ("Hello RocketMQ " + i).getBytes(RemotingHelpe.DEFAULT_CHARSET));
/// Set some properties .
msg.putUserProperty("a", String.valueOf(i));
msg.putUserProperty("b", "hello");
```
代码中这个消息就有了两个特殊的属性值 a 和 b ，我们用类似 SQL 表达式的方式对消息进行过滤，用法如下(目前只支持在 PushConsumer 中实现这种过滤) :
```java
DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("pleas
rename_unique_group_name_4");
// only subsribe messages have property a , also a >=0 and a <= 3 
consumer.subscribe("TopicTest", MessageSelector.bySql("a between 0 and 3 ");
consumer.registerMessageListener(new MessageListenerConcurrently(){
	@Override 
    public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs , ConsumeConcurrentlyContext context){
		return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
    }
});
consumer.start();
```
类似 SQL 的过滤表达式,支持如下语法:

-  数字对比,比如>、>=、<、<=、 BETWEEN、= 
-  字符串对比,比如=、<>、IN; 
-  IS NULL or IS NOT NULL 
-  逻辑符号AND、OR、NOT。
支持的数据类型: 
-  数字型,比如123、3.1415; 
-  字符型,比如'abc'、注意必须用单引号; 
-  NULL,这个特殊字符; 
-  布尔型, TRUEOrFALSE。 

SQL 表达式方式的过滤需要 Broker先读出消息里的属性内容,然后做SQL计算,增大磁盘压力,没有 Tag 方式高效。
### Filter Server方式过滤
Filter Server 种比 SQL 达式更灵活的过滤方式，允许用户自定义 Java 函数，根据 Java 逻辑对消息进行过滤。
使用 Filter Server , 首先要在启动 Broker 在配置文件里加上 filterServerNums = 3 这样的配置, Broker 启动的时, 就会在本机启动 3 个 Filter Server 进程。Filter Server 类似一个 RocketMQ 的 Consumer 进程，它从本机 Broker 获取消息，然后根据用户上传过来的 Java 进行过滤，过滤后的消息再传给远端的 Consumer。这种方式会占用很多 Broker 机器的 CPU 资源，要根据实际情况谨慎使用。上传的 java 代码也要经过检查，不能有申请大内存、创建线程等这样的操作，否则容易造成 Broker 服务器宕机。实现过滤逻辑的示例, 如代码清单所示。
```java
public class MessageFilterImpl implements MessageFilter {
    @Override
    public boolean match(MessageExt messageExt, FilterContext filterContext) {
        String property = messageExt.getUserProperty("SequenceId");
        if (property != null) {
            int id = Integer.parseInt(property);
            if ((id % 3) == 0 && (id > 10)) {
                return true;
            }
        }
        return false;
    }
}
```
上面代码实现了过滤逻辑，它是根据消息的“Sequenceld”这个属性来过滤的，其实不一定要根据消息属性来过滤，也可以根据消息体的内容或其他特征过滤，如代码清单所示。
```java
public static void main(String[] args) throws InterruptedException, MQClientException {
    DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("Consumer-GroupNamecc4");
    // 使用Java代码,在服务器做消息过滤
    String filterCode = MixAll.file2String("/home/admin/MessageFilterImpl.java");
    consumer.subscribe("TopicFilter7", "com.alibaba.rocketmqexample.filter.MessageFilterImpl", filterCode);
    consumer.registerMessageListener(new MessageListenerConcurrently() {
        @Override
        public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs, ConsumeConcurrentlyContext context) {
            System.out.println(Thread.currentThread().getName() + " Receive New Messages: " + msgs);
            return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
        }
    });
}
```
在使用 Filter Server 的 Consumer 例子中，主要是把实现过滤逻辑的类作为参数传到 Broker 端， Broker 端的 Filter Server 会解析这个类，然后根据 match 函数里的逻辑进行过滤。
## 提高 Consumer 处理能力
当 Consumer 的处理速度眼不上消息的产生速度，会造成越来越多的消息积压，这个时候首先查看消费逻辑本身有没有优化空间，除此之外还有三种方法可以提高 Consumer 的处理能力。
(1) 提高消费并行度
在同一个 Consumer Group 下 ( Clustering方式), 可以通过增加 Consumer 实例的数量来提高并行度, 通过加机器, 或者在已有机器中启动多个 Consumer 进程都可以增加 Consumer实例数。注意总的 Consumer 数量不要超过 Topic 下 Read Queue 数量, 超过的 Consumer 实例接收不到消息。此外,通过提高单个 Consumer 实例中的并行处理的线程数, 可以在同一个 Consumer 内增加并行度来提高吞吐量(设置方法是修改 consumeThreadMin 和 consumeThreadMax)。
(2) 以批量方式进行消费
某些业务场景下,多条消息同时处理的时间会大大小于逐个处理的时间总和,比如消费消息中涉及update 某个数据库,一次 update 10 条的时间会大大小于 10 次 update 1 条数据的时间。这时可以通过批量方式消费来提高消费的吞吐量。实现方法是设置 Consumer的 consumeMessageBatchMaxSize 这个参数, 默认是 1 , 如果设置为N,在消息多的时候每次收到的是个长度为N的消息链表。
(3) 检测延时情况, 跳过非重要消息
Consumer在消费的过程中,如果发现由于某种原因发生严重的消息堆积短时间无法消除堆积,这个时候可以选择丢弃不重要的消息,使 Consumer 尽快追上 Producer 的进度,如代码清单所示。
```java
consumer.registerMessageListener(new MessageListenerConcurrently() {
    @Override
    public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs, ConsumeConcurrentlyContext context) {
        long offset = msgs.get(0).getQueueOffset();
        String maxOffset = msgs.get(0).getProperty("MAX_OFFSET");
        long diff = Long.parseLong(maxOffset) - offset;
        if (diff > 90000) {
            // 不处理直接返回成功
            return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
        } else {
            System.out.println("正常消费消息");
            return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
        }
    }
});
```
如代码所示,当某个队列的消息数堆积到 90000 条以上,就直接丢弃,以便快速追上发送消息的进度。
### Consumer 的负载均衡
节中讲到,想要提高 Consumer的处理速度,可以启动多个 Consumer 并发处理,这个时候就涉及如何在多个 Consumer之间负载均衡的问题,接下来结合源码分析 Consumer的负载均衡实现。
要做负载均衡,必须知道一些全局信息,也就是一个 Consumer Group里到底有多少个 Consumer,知道了全局信息,才可以根据某种算法来分配,比如简单地平均分到各个 Consumer在 RocketMQ 中,负载均衡或者消息分配是在 Consumer端代码中完成的, Consumer从 Broker 处获得全局信息, 然后自己做负载均衡,只处理分给自己的那部分消息。
#### DefaultMQPushConsumer的负载均衡
DefaultMQPushConsumer 的负载均衡过程不需要使用者操心,客户端程序会自动处理,每个 DefaultMQPush Consumer启动后,会马上会触发一个 doRebalance 动作; 而且在同一个 Consumer Group 里加入新的 DefaultMQPushConsumer 时,各个 Consumer 都会被触发 doRebalance 动作。
默认用的是 AllocateMessageQueueAveragely。AllocateMessageQueueAveragely 平均分配策略; AllocateMessageQueueAveragelyByCircle 环形分配策; AllocateMessageQueueByConfig 手动配置分配策略; AllocateMessageQueueByMachineRoom 机房分配策略。
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240309001250.png#id=AyjnX&originHeight=241&originWidth=1240&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
负载均衡的结果与 Topic的 Message Queue 数量,以及 Consumer Group 里的 Consumer 的数量有关。负载均衡的分配粒度只到 Message Queue, 把 Topic 下的所有 Message Queue 分配到不同的 Consumer 中, 所以 Message Queue 和 Consumer 的数量关系,或者整除关系影响负载均衡结果。
以 AllocateMessageQueueAveragely 策略为例,如果创建 Topic 的时候,把 MessageQueue 数设为 3 ,当 Consumer 数量为 2 的时候,有一个 Consumer需要处理 Topic 三分之二的消息, 另一个处理三分之一的消息; 当 Consumer 数量为 4 的时候, 有一个 Consumer 无法收到消息, 其他 3 个 Consumer各处理 Topic 三分之一的消息。可见 Message Queue 数量设置过小不利于做负载均衡, 通常情况下,应把一个 Topic的 Message Queue 数设置为 16。
#### DefaultMQPulLConsumer的负载均衡
Pull Consumer 可以看到所有的 Message Queue, 而且从哪个 Message Queue 读取消息,读消息时的 Offset 都由使用者控制, 使用者可以实现任何特殊方式的负载均衡。
DefaultMQPullConsumer 有两个辅助方法可以帮助实现负载均衡, 一个是 registerMessageQueueListener 函数,如代码清单所示。registerMessageQueueListener 函数在有 Consumer 加人或退出时被触发。
```java
DefaultMQPullConsumer consumer = new DefaultMQPullConsumer("please_rename_unique_group_name");
consumer.registerMessageQueueListener("", new MessageQueueListener() {
    @Override
    public void messageQueueChanged(String s, Set<MessageQueue> set, Set<MessageQueue> set1) {
    // todo 
    }
});
```
另一个辅助工具是 MQPullConsumerScheduleService 类，使用这个 Class 类似使用 DefaultMQPushConsumer ，但是它把 Pull 消息的 主动性留给了使用者，如代码清单所示。
```java
public static void main(String[] args) throws MQClientException {
    final MQPullConsumerScheduleService scheduleService = new MQPullConsumerScheduleService("please_rename_unique_group_name");
    scheduleService.getDefaultMQPullConsumer().setNamesrvAddr(RocketMqConfig.NAMESRV_ADDR);
    scheduleService.setMessageModel(MessageModel.CLUSTERING);
    scheduleService.registerPullTaskCallback("testPullConsumer", new PullTaskCallback() {
        @Override
        public void doPullTask(MessageQueue messageQueue, PullTaskContext pullTaskContext) {
            MQPullConsumer consumer = pullTaskContext.getPullConsumer();
            try {
                long offset = consumer.fetchConsumeOffset(messageQueue, false);
                if (offset < 0) {
                    offset = 0;
                }
                PullResult pullResult = consumer.pull(messageQueue, "*", offset, 32);
                System.out.printf("%s%n", +offset + "\t" + messageQueue + "\t" + pullResult);
                switch (pullResult.getPullStatus()) {
                    case FOUND:
                        break;
                    case NO_NEW_MSG:
                        break;
                    case NO_MATCHED_MSG:
                        break;
                    case OFFSET_ILLEGAL:
                        break;
                    default:
                        break;
                }
                consumer.updateConsumeOffset(messageQueue, pullResult.getNextBeginOffset());
                pullTaskContext.setPullNextDelayTimeMillis(1000);
            } catch (MQClientException | InterruptedException | RemotingException | MQBrokerException e) {
                e.printStackTrace();
            }
        }
    });
    scheduleService.start();
}
```
然后我们看一看在 MQPullConsumerScheduleService 类的实现里, 实现负载均衡的代码, 如下代码清单所示。
```java
class MessageQueueListenerImpl implements MessageQueueListener {
    MessageQueueListenerImpl() {
    }

    public void messageQueueChanged(String topic, Set<MessageQueue> mqAll, Set<MessageQueue> mqDivided) {
        MessageModel messageModel = MQPullConsumerScheduleService.this.defaultMQPullConsumer.getMessageModel();
        switch(messageModel) {
            case BROADCASTING:
                MQPullConsumerScheduleService.this.putTask(topic, mqAll);
                break;
            case CLUSTERING:
                MQPullConsumerScheduleService.this.putTask(topic, mqDivided);
        }
    }
}
```
从源码中可以看出, 用户通过更改 MessageQueueListenerImpl 的实现来做自己的负载均衡策略。
## 提高 Producer 的发送速度
发送一条消息出去要经过三步, 一是客户端发送请求到服务器, 二是服务器处理该请求, 三是服务器向客户端返回应答, 一次消息的发送耗时是上述三个步骤的总和。在一些对速度要求高,但是可靠性要求不高的场景下, 比如日志收集类应用, 可以采用 Oneway 方式发送, Oneway 方式只发送请求不等待应答, 即将数据写入客户端的 Socket 缓冲区就返回, 不等待对方返回结果, 用这种方式发送消息的耗时可以缩短到微秒级。
另一种提高发送速度的方法是增加 Producer 的并发量,使用多个 Producer 同时发送, 我们不用担心多 Producer 同时写会降低消息写磁盘的效率 RocketMQ 引入了一个并发窗口,在窗口内消息可以并发地写入 DirectMem 中, 然后异步地将连续一段无空洞的数据刷入文件系统当中。顺序写 CommitLog 可让 RocketMQ 无论在 HDD 还是 SSD 磁盘情况下都能保持较高的写入性能。目前在阿里内部经过调优的服务器上, 写入性能达到90万+的TPS, 我们可以参考这个数据进行系统优化。
在 Linux 操作系统层级进行调优,推荐使用EXT4文件系统, IO调度算法使用 deadline 算法。
如图所示,EXT4 创建/删除文件的性能比 EXT3 及其他文件系统要好, RocketMQ的 Commitlog会有频繁的创建/删除动作。
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240309001334.png#id=sWTFb&originHeight=314&originWidth=553&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
另外, IO 调度算法也推荐调整为 deadline。deadline 算法大致思想如下 : 实现四个队列,其中两个处理正常的 read 和 write 操作, 另外两个处理超时的 read 和 write 操作。正常的 read 和 write 队列中, 元素按扇区号排序,进行正常的 IO 合并处理以提高吞吐量。因为 IO 请求可能会集中在某些磁盘位置,这样会导致新来的请求一直被合并,可能会有其他磁盘位置的IO请求被饿死。超时的 read 和 write 的队列中, 元素按请求创建时间排序, 如果有超时的请求出现, 就放进这两个队列,调度算法保证超时(达到最终期限时间)的队列中的 IO 请求会优先被处理。
