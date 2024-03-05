# RocketMQ 不同类型的生产者

> 《RocketMQ实战与原理解析》

## DefaultMQProducer 的使用
发送消息要经过五个步骤：
1）设置 Producer 的 GroupName。
2）设置 InstanceName，当一个 Jvm 需要启动多个 Producer 的时候， 通过设置不同的 InstanceName 来区分，不设置的话系统使用默认名称“DEFAULT”。
3）设置发送失败重试次数，当网络出现异常的时候，这个次数影响消息的重复投递次数。想保证不丢消息，可以设置多重试几次。
4）设置 NameServer 地址。
5）组装消息并发送。消息的发送有同步和异步两种方式，上面的代码使用的是异步方式。
在第 2 章的例子中用的是同步方式。消息发送的返回状态有如下四种： FLUSH_DISK_TIMEOUT、FLUSH_SLAVE_TIMEOUT、SLAVE_NOT_AVAILABLE、SEND_OK，不同状态在不同的刷盘策略和同步策略的配置下含义是不同的。

-  FLUSH_ DISK_TIMEOUT：表示没有在规定时间内完成刷盘（需要 Broker 的刷盘策略被设置成 SYNC_FLUSH 才会报这个错误）。 
-  FLUSH_SLAVE_TIMEOUT：表示在主备方式下，并且 Broker 被设置成 SYNC_MASTER 方式，没有在设定时间内完成主从同步。 
-  SLAVE_NOT_AVAILABLE：这个状态产生的场景和 FLUSH_SLAVE_TIMEOUT 类似，表示在主备方式下，并且 Broker 被设置成 SYNC_MASTER，但是没有找到被配置成 Slave 的 Broker。 
-  SEND_OK：表示发送成功，发送成功的具体含义，比如消息是否已经被存储到磁盘？ 

消息是否被同步到了 Slave 上？消息在 Slave 上是否被写入磁盘？ 需要结合所配置的刷盘策略、主从策略来定。这个状态还可以简单理解为，没有发生上面列出的三个问题状态就是 SEND_OK。写一个高质量的生产者程序，重点在于对发送结果的处理，要充分考虑各种异常，写清对应的处理逻辑。
### Demo
```java
public class DefaultMQProducerDemo {


    public static void main(String[] args) throws MQClientException, InterruptedException {
        DefaultMQProducer producer = new DefaultMQProducer("please_rename_unique_group_name");
        producer.setInstanceName("instance1");
        producer.setRetryTimesWhenSendFailed(3);
        producer.setNamesrvAddr("127.0.0.1:9876");
        producer.start();
        for (int i = 0; i < 10; i++) {
            try {
                Message msg = new Message("TopicTest", "TagA", ("Hello RocketMQ " + i).getBytes(RemotingHelper.DEFAULT_CHARSET));
                producer.send(msg, new SendCallback() {
                    @Override
                    public void onSuccess(SendResult sendResult) {
                        System.out.printf("%s %n", sendResult);
                        sendResult.getSendStatus();
                    }

                    @Override
                    public void onException(Throwable e) {
                        e.printStackTrace();
                    }
                });
            } catch (Exception e) {
                e.printStackTrace();
                Thread.sleep(1000);
            }
        }
        producer.shutdown();
    }
}
```
## TransactionMQProducer 发送事务消息
RocketMQ 的事务消息，是指发送消息事件和其他事件需要同时成功或同时失败。比如银行转账，A银行的某账户要转一万元到B银行的某账户。A银行发送“B银行账户增加一万元”这个消息，要和“从A银行账户扣除一万元”这个操作同时成功或者同时失败。
RocketMQ 采用两阶段提交的方式实现事务消息，TransactionMQProducer 处理上面情况的流程是，先发一个“准备从B银行账户增加一万元”的消息，发送成功后做从A银行账户扣除一万元的操作，根据操作结果是否成功，确定之前的“准备从B银行账户增加一万元” 的消息是做 commit 还是 rollback，具体流程如下：
1）发送方向 RocketMQ 发送“待确认”消息。
2）RocketMQ 将收到的“待确认”消息持久化成功后，向发送方回复消息已经发送成功，此时第一阶段消息发送完成。
3）发送方开始执行本地事件逻辑。
4）发送方根据本地事件执行结果向 RocketMQ 发送二次确认(Commit 或是 Rollback)消息，
RocketMQ 收到 Commit 状态则将第一阶段消息标记为可投递，阅方将能够收到该消息；收到 Rollback 状态则删除第一阶段的消息，订阅方接收不到该消息。
5）如果出现异常情况，步骤 4）提交的二次确认最终未到达 RocketMQ，服务器在经过固定时间段后将对“待确认”消息发起回查请求。
6）发送方收到消息回查请求后(如果发送一阶段消息的 Producer 不能工作，回查请求将被发送到和 Producer 在同一个 Group 里的其他 Producer)，
通过检查对应消息的本地事件执行结果返回 Commit 或 Roolback 状态。
7）RocketMQ 收到回查请求后，按照步骤 4）的逻辑处理。
上面的逻辑似乎很好地实现了事务消息功能，它也 RocketMQ 之前的版本实现事务消息的逻辑。但是因为 RocketMQ 依赖将数据顺序写到磁盘这个特征来提高性能，步骤 4）却需要更改第一阶段消息的状态，这样会造成磁盘 Catch 的脏页过多，降低系统的性能。所以 RocketMQ 在 4.x 的版本中将这部分功能去除。系统中的一些上层 Class 都还在，用户可以根据实际需求实现自己的事务功能。
客户端有三个类来支持用户实现事务消息，
第一个类是 LocalTransactionExecuter，用来实例化步骤 3）的逻辑，根据情况返回 LocalTransactionState.ROLLBACK_MESSAGE 或者 LocalTransactionState.COMMIT_MESSAGE 状态。
第二个类是 TransactionMQProducer，它的用法和 DefaultMQProducer 类似，要通过它启动一个 Producer 并发消息，但是比 DefaultMQProducer 多设置本地事务处理函数和回查状态函数。
第三个类是 TransactionCheckListener，实现步骤 5）中 MQ 服务器的回查请求，返回 LocalTransactionState.ROLLBACK_ MESSAGE 或者 LocalTransactionState.COMMIT_ MESSAGE。
### Demo
消息体
```java
@Data
@AllArgsConstructor
public class UserTransferMessage {

    /**
     * 流水id
     */
    private Long flowId;

    /**
     * 转账发起人用户
     */
    private String fromUser;

    /**
     * 转账接收人用户
     */
    private String toUser;

    /**
     * 转账金额
     */
    private Long amount;

}
```
#### rocketmq-client 3.* 版本
实现LocalTransactionExecuter
```java
@Slf4j
public class TransactionExecuterImpl implements LocalTransactionExecuter {

    public static ConcurrentHashMap<Long, Integer> localTrans = new ConcurrentHashMap<>();

    @Override
    public LocalTransactionState executeLocalTransactionBranch(final Message msg, final Object arg) {
        String transactionMqMessageJson = new String(msg.getBody());
        log.info("接收到事务消息 transactionMqMessageJson = {}", transactionMqMessageJson);
        UserTransferMessage userTransferMessage = GsonUtil.GSON.fromJson(transactionMqMessageJson, UserTransferMessage.class);
        log.info("开始执行本地事务");
        Long transactionMQMessageId = userTransferMessage.getFlowId();
        /*
            随机设置事务执行结果
            transactionMQMessageId % 2 == 0 , 执行完成, 成功
            transactionMQMessageId % 2 == 1 , 执行完成, 失败
            transactionMQMessageId % 2 == 2 , 未知
         */
        localTrans.put(userTransferMessage.getFlowId(), (int) (transactionMQMessageId % 3));
        int result = localTrans.get(userTransferMessage.getFlowId());
        LocalTransactionState state;
        if (result == 0) {
            state = LocalTransactionState.COMMIT_MESSAGE;
        } else if (result == 1) {
            state = LocalTransactionState.ROLLBACK_MESSAGE;
        } else {
            state = LocalTransactionState.UNKNOW;
        }
        log.info("执行本地事务完成 state = {}", state);
        return state;
    }
}
```
实现TransactionCheckListener
```java
@Slf4j
public class TransactionCheckListenerImpl implements TransactionCheckListener {

    @Override
    public LocalTransactionState checkLocalTransactionState(MessageExt messageExt) {
        String transactionMqMessageJson = new String(messageExt.getBody());
        log.info("校验本地事务的结果, transactionMqMessageJson = {}", transactionMqMessageJson);
        /*
            随机设置事务执行结果
            randomInt % 2 == 0 , 执行完成, 成功
            randomInt % 2 == 1 , 执行完成, 失败
            randomInt % 2 == 2 , 未知
         */
        long randomInt = RandomUtil.RANDOM.nextInt(Integer.MAX_VALUE) % 3;
        log.info("TransactionCheckListenerImpl, randomInt = {}", randomInt);
        UserTransferMessage userTransferMessage = GsonUtil.GSON.fromJson(transactionMqMessageJson, UserTransferMessage.class);
        TransactionExecuterImpl.localTrans.put(userTransferMessage.getFlowId(), (int) randomInt);
        if (randomInt == 0) {
            return LocalTransactionState.COMMIT_MESSAGE;
        } else if (randomInt == 1) {
            return LocalTransactionState.ROLLBACK_MESSAGE;
        }
        return LocalTransactionState.UNKNOW;
    }
}
```
利用TransactionMQProducer发送消息
```java
@Slf4j
public class TransactionMQProducerDemo {

    public static final String TOPIC_NAME = "TransactionMQProducerTopic";

    public static void main(String[] args) throws MQClientException, InterruptedException {
        //初始化transactionCheckListener
        TransactionCheckListener transactionCheckListener = new TransactionCheckListenerImpl();
        TransactionExecuterImpl transcationExuctor = new TransactionExecuterImpl();
        //创建事务TransactionMQProducer
        TransactionMQProducer producer = new TransactionMQProducer("TransactionMQProducerGroup");
        producer.setNamesrvAddr(MqConfig.NAMESRV_ADDR);
        //设置事务监听
        producer.setTransactionCheckListener(transactionCheckListener);
        producer.start();
        for (int i = 0; i < 1; i++) {
            try {
                long flowId = RandomUtil.RANDOM.nextInt(Integer.MAX_VALUE);
                long amount = RandomUtil.RANDOM.nextInt(Integer.MAX_VALUE);
                UserTransferMessage userTransferMessage = new UserTransferMessage(flowId, "A", "B", amount);
                Message msg = new Message(TOPIC_NAME, "TagA", "KEY" + i,
                        GsonUtil.GSON.toJson(userTransferMessage).getBytes(RemotingHelper.DEFAULT_CHARSET));
                SendResult sendResult = producer.sendMessageInTransaction(msg, transcationExuctor, null);
                log.info("消息发送完成 sendResult = {}", sendResult);
                Thread.sleep(10);
            } catch (MQClientException | UnsupportedEncodingException e) {
                e.printStackTrace();
            }
        }
        Thread.currentThread().join();
        producer.shutdown();

    }
}
```
#### rocketmq-client 4.* 版本
```java
@Slf4j
public class TransactionListenerImpl implements TransactionListener {

    private AtomicInteger transactionIndex = new AtomicInteger(0);

    private ConcurrentHashMap<String, Integer> localTrans = new ConcurrentHashMap<>();

    @Override
    public LocalTransactionState executeLocalTransaction(Message msg, Object arg) {
        log.info("msgBody = {}", new String(msg.getBody()));
        int value = transactionIndex.getAndIncrement();
        int status = value % 3;
        log.info("status = {}", status);
        localTrans.put(msg.getTransactionId(), status);
        return LocalTransactionState.UNKNOW;
    }

    @Override
    public LocalTransactionState checkLocalTransaction(MessageExt msg) {
        log.info("msgBody = {}", new String(msg.getBody()));
        Integer status = localTrans.get(msg.getTransactionId());
        if (null != status) {
            switch (status) {
                case 0:
                    return LocalTransactionState.UNKNOW;
                case 1:
                    return LocalTransactionState.COMMIT_MESSAGE;
                case 2:
                    return LocalTransactionState.ROLLBACK_MESSAGE;
            }
        }
        return LocalTransactionState.COMMIT_MESSAGE;
    }
}
```
```java
@Slf4j
public class TransactionProducer {

    public static final String TOPIC_NAME = "TransactionMQProducerTopic_v4";

    public static void main(String[] args) throws MQClientException, InterruptedException {
        TransactionListener transactionListener = new TransactionListenerImpl();
        TransactionMQProducer producer = new TransactionMQProducer("TransactionMQProducerGroupv4");
        ExecutorService executorService = new ThreadPoolExecutor(2, 5, 100, TimeUnit.SECONDS, new ArrayBlockingQueue<Runnable>(2000), new ThreadFactory() {
            @Override
            public Thread newThread(Runnable r) {
                Thread thread = new Thread(r);
                thread.setName("client-transaction-msg-check-thread");
                return thread;
            }
        });
        producer.setNamesrvAddr(MqConfig.NAMESRV_ADDR);
        producer.setExecutorService(executorService);
        producer.setTransactionListener(transactionListener);
        producer.start();

        String[] tags = new String[]{"TagA", "TagB", "TagC", "TagD", "TagE"};
        for (int i = 0; i < 10; i++) {
            try {
                Message msg =
                        new Message(TOPIC_NAME, tags[i % tags.length], "KEY" + i,
                                ("Hello RocketMQ " + i).getBytes(RemotingHelper.DEFAULT_CHARSET));
                SendResult sendResult = producer.sendMessageInTransaction(msg, null);

                log.info("sendResult = {}", sendResult);

                Thread.sleep(10);
            } catch (MQClientException | UnsupportedEncodingException e) {
                e.printStackTrace();
            }
        }

        for (int i = 0; i < 100000; i++) {
            Thread.sleep(1000);
        }
        producer.shutdown();
    }
}
```
**使用约束**
(1)事务消息不支持调度和批处理。
(2)为了避免单个消息多次检查,导致一半队列消息积累,我们有限的数量检查单个消息默认15次,但用户可以改变这个极限,改变“transactionCheckMax”参数的配置代理,如果一个消息一直在检查“transactionCheckMax”时代,代理会丢弃该消息和打印一个错误日志默认在同一时间。用户可以通过重写“AbstractTransactionCheckListener”类来更改此行为。
(3)事务消息将在代理配置中的参数“transactionTimeout”确定的一段时间后被检查。用户也可以通过设置用户属性“CHECK_IMMUNITY_TIME_IN_SECONDS”来改变这个限制。在发送事务性消息时，该参数优先于“transactionMsgTimeout”参数。
(4)事务消息可能会被检查或使用多次。
(5)重新提交到用户目标主题的已提交消息可能会失败。目前，它取决于日志记录。高可用性是由RocketMQ本身的高可用性机制保证的。如果您希望确保事务消息不会丢失，并且保证事务的完整性，建议使用同步双写。机制。
(6)事务性消息的生产者id不能与其他类型消息的生产者id共享。与其他类型的消息不同，事务性消息允许向后查询。MQ服务器通过客户机的生产者id查询客户机。
## 发送延时消息
发送延迟消息 RocketMQ 支持发送延迟消息， Broker 收到这类消息后，延迟一段时间再处理，使消息在规定的一段时间后生效。
延迟消息的使用方法是在创建 Message 对象时，调用 setDelayTimeLevel(int level) 方法设置延迟时间，然后再把这个消息发送出去。
目前延迟的时间不支持任意设置，仅支持预设值的时间长度（1s/5s/10s/30s/1m/2m/3m/4m/5m/6m/7m/8m/9m/10m/20m/30m/1h/2h）。
比如 setDelayTimeLevel(3) 表示延迟10s
```java
Message msg = new Message("TopicTest", "TagA", ("Hello RocketMQ " + i).getBytes(RemotingHelper.DEFAULT_CHARSET));
msg.setDelayTimeLevel(3);
```
## 自定义发送规则
自定义消息发送规则一个 Topic 会有多个 Message Queue， 如果使用 Producer 的默认配置，这个 Producer 会轮流向各个 Message Queue 发送消息。
Consumer 在消费消息的时候，会根据负载均衡策略，消费被分配到的 Message Queue，如果不经过特定的设置，某条消息被发往哪个 Message Queue，被哪个 Consumer 消费是未知的。
如果业务需要我们把消息发送到指定的 Message Queue 里，比如把同一类型的消息都发往相同的 Message Queue，该怎么办呢？
可以用 MessageQueueSelector，发送消息的时候，把 MessageQueueSelector 的对象作为参数，使用 public SendResult send(Message msg, MessageQueueSelector selector, Object arg) 函数发送消息即可。在 MessageQueueSelector 的实现中，根据传入的 Object 参数，或者根据 Message 消息内容确定把消息发往那个 Message Queue，返回被选中的 Message Queue。
```java
public class OrderMessageQueueSelector implements MessageQueueSelector {

    private Random random = new Random();

    @Override
    public MessageQueue select(List<MessageQueue> mqs, Message msg, Object orderKey) {
        int size = mqs.size();
        int index = random.nextInt() % size;
        return mqs.get(index);
    }
}
```
```java
// 发送时设置选择器
try {
    Message msg = new Message("TopicTest", "TagA", ("Hello RocketMQ " + i).getBytes(RemotingHelper.DEFAULT_CHARSET));
    msg.setDelayTimeLevel(3);
    producer.send(msg, messageQueueSelector, new SendCallback() {
        @Override
        public void onSuccess(SendResult sendResult) {
            System.out.printf("%s %n", sendResult);
            SendStatus sendStatus = sendResult.getSendStatus();
            log.info("sendStatus = {}", sendStatus);
        }

        @Override
        public void onException(Throwable e) {
            e.printStackTrace();
        }
    });
} catch (Exception e) {
    e.printStackTrace();
}
```
## 发送顺序消息
消息有序指的是一类消息消费时，能按照发送的顺序来消费。例如：一个订单产生了三条消息分别是订单创建、订单付款、订单完成。消费时要按照这个顺序消费才能有意义，但是同时订单之间是可以并行消费的。RocketMQ可以严格的保证消息有序。
顺序消息分为全局顺序消息与分区顺序消息，全局顺序是指某个Topic下的所有消息都要保证顺序；部分顺序消息只要保证每一组消息被顺序消费即可。

- 全局顺序 对于指定的一个 Topic，所有消息按照严格的先入先出（FIFO）的顺序进行发布和消费。 适用场景：性能要求不高，所有的消息严格按照 FIFO 原则进行消息发布和消费的场景
- 分区顺序 对于指定的一个 Topic，所有消息根据 sharding key 进行区块分区。 同一个分区内的消息按照严格的 FIFO 顺序进行发布和消费。 Sharding key 是顺序消息中用来区分不同分区的关键字段，和普通消息的 Key 是完全不同的概念。 适用场景：性能要求高，以 sharding key 作为分区字段，在同一个区块中严格的按照 FIFO 原则进行消息发布和消费的场景。

RocketMQ可以严格的保证消息有序。但这个顺序，不是全局顺序，只是分区（queue）顺序。要全局顺序只能一个分区。
```java
@Slf4j
public class OrderedProducerDemo {
    public static void main(String[] args) throws Exception {
        //Instantiate with a producer group name.
        DefaultMQProducer producer = new DefaultMQProducer("ordered_producer_group");
        //Launch the instance.
        producer.setNamesrvAddr(RocketMqConfig.NAMESRV_ADDR);
        producer.start();

        String[] tags = new String[]{"TagA", "TagB", "TagC", "TagD", "TagE"};
        for (int i = 0; i < 100; i++) {
            int orderId = i % 10;
            //Create a message instance, specifying topic, tag and message body.
            Message msg = new Message("OrderedProducerTopic", tags[i % tags.length], "KEY" + i,
                    ("Hello RocketMQ " + i).getBytes(RemotingHelper.DEFAULT_CHARSET));
            SendResult sendResult = producer.send(msg, (mqs, msg1, arg) -> {
                Integer id = (Integer) arg;
                int index = id % mqs.size();
                return mqs.get(index);
            }, orderId);
            log.info("sendResult = {}", sendResult);
        }
        //server shutdown
        producer.shutdown();
    }
}
```
```java
@Slf4j
public class OrderedConsumerDemo {

    public static void main(String[] args) throws Exception {
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("ordered_consumer_group");

        consumer.setNamesrvAddr(RocketMqConfig.NAMESRV_ADDR);

        consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);

        consumer.subscribe("OrderedProducerTopic", "TagA || TagC || TagD");

        consumer.registerMessageListener(new MessageListenerOrderly() {

            AtomicLong consumeTimes = new AtomicLong(0);

            @Override
            public ConsumeOrderlyStatus consumeMessage(List<MessageExt> msgs,
                                                       ConsumeOrderlyContext context) {
                context.setAutoCommit(false);
                log.info("{} Receive New Messages: {}", Thread.currentThread().getName(), msgs);
                this.consumeTimes.incrementAndGet();
                if ((this.consumeTimes.get() % 2) == 0) {
                    return ConsumeOrderlyStatus.SUCCESS;
                } else if ((this.consumeTimes.get() % 3) == 0) {
                    return ConsumeOrderlyStatus.ROLLBACK;
                } else if ((this.consumeTimes.get() % 4) == 0) {
                    return ConsumeOrderlyStatus.COMMIT;
                } else if ((this.consumeTimes.get() % 5) == 0) {
                    context.setSuspendCurrentQueueTimeMillis(3000);
                    return ConsumeOrderlyStatus.SUSPEND_CURRENT_QUEUE_A_MOMENT;
                }
                return ConsumeOrderlyStatus.SUCCESS;

            }
        });

        consumer.start();

        System.out.printf("Consumer Started.%n");
    }
}
```
## Demo代码的相关依赖配置
Maven依赖
```xml
<dependency>
    <groupId>com.alibaba.rocketmq</groupId>
    <artifactId>rocketmq-client</artifactId>
    <version>3.6.2.Final</version>
</dependency>
<dependency>
    <groupId>com.google.code.gson</groupId>
    <artifactId>gson</artifactId>
    <version>2.8.6</version>
</dependency>
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <version>1.18.8</version>
</dependency>

<dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>slf4j-log4j12</artifactId>
    <version>1.7.21</version>
</dependency>

<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-lang3</artifactId>
    <version>3.7</version>
</dependency>

<dependency>
    <groupId>commons-collections</groupId>
    <artifactId>commons-collections</artifactId>
    <version>3.2.2</version>
</dependency>
```
日志配置
```properties
log4j.rootLogger=INFO, stdout
log4j.appender.stdout=org.apache.log4j.ConsoleAppender
log4j.appender.stdout.layout=org.apache.log4j.PatternLayout
log4j.appender.stdout.layout.ConversionPattern=%d %p [%c] - %m%n
```
