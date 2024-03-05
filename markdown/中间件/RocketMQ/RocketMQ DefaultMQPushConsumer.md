# RocketMQ DefaultMQPushConsumer

> 《RocketMQ实战与原理解析》

## DefaultMQPushConsumer的使用
DefaultMQPushConsumer，由系统控制读取操作，收到消息后自动调用传入的处理方法来处理；使用 DefaultMQPushConsumer 主要是设置好各种参数和传入处理消息的函数。系统收到消息后自动调用处理函数来处理消息，自动保存 Offset，而且加入新的DefaultMQPushConsumer 后会自动做负载均衡。
### RocketMQ的消息模式
RocketMQ支持两种消息模式： Clustering 和 Broadcasting。
·在 Clustering 模式下， 同一个 ConsumerGroup（ GroupName 相同） 里的每个 Consumer 只消费所订阅消息的一部分内容， 同一个 ConsumerGroup 里所有的 Consumer 消费的内容合起来才是所订阅 Topic 内容的整体， 从而达到负载均衡的目的。 ·
在 Broadcasting 模式下， 同一个 ConsumerGroup 里的每个 Consumer 都能消费到所订阅Topic 的全部 消息， 也就是 一个消息会被 多次分发，被多个 Consumer 消费。
### 使用Demo
```java
/**
 * DefaultMQPushConsumer，由系统控制读取操作，收到消息后自动调用传入的处理方法来处理；
 * 使用 DefaultMQPushConsumer 主要是设置好各种参数和传入处理消息的函数。
 * 系统收到消息后自动调用处理函数来处理消息，自动保存 Offset，而且加入新的 DefaultMQPushConsumer 后会自动做负载均衡。
 *
 */
public class DefaultMQPushConsumerDemo {

    public static void main(String[] args) throws MQClientException {
        // Consumer 的 GroupName 用于把多个 Consumer 组织到一起，提高并发处理能力，GroupName 需要和消息模式（ MessageModel） 配合使用。
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("please_rename_unique_group_name");
        // NameServer 的地址和端口号，可以填写多个，用分号隔开，达到消除单点故障的目的，比如“ip1：port；ip2：port；ip3：port”。
        consumer.setNamesrvAddr("127.0.0.1:9876");
        /* Specify where to start in case the specified Consumer group is a brand new one. */
        consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);
        consumer.setMessageModel(MessageModel.BROADCASTING);
        /* Subscribe one more more Topics to consume. */
        // Topic 名称用来标识消息类型，需要提前创建。
        // 如果不需要消费某个 Topic 下的所有消息，可以通过指定消息的 Tag 进行消息过滤，
        // 比如：Consumer.subscribe（"TopicTest"，"tag1||tag2||tag3"），表示这个 Consumer 要消费“TopicTest”下带有tag1或tag2或tag3的消息
        // （ Tag 是在发送消息时设置的标签）。在填写 Tag 参数的位置， 用 null 或者“*” 表示要消费这个 Topic 的所有消息。
        consumer.subscribe("TopicTest", "*");
        /* * Register callback to execute on arrival of Messages fetched from brokers. */
        consumer.registerMessageListener(
                (MessageListenerConcurrently) (msgs, context) -> {
                    System.out.printf(Thread.currentThread().getName() + " Receive New Messages: " + msgs + "%n");
                    System.out.println();
                    return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
                });
        /* * Launch the Consumer instance.*/
        consumer.start();
    }

}
```
## DefaultMQPushConsumer 的处理流程
DefaultMQPushConsumer 主要功能实现在 DefaultMQPushConsumerImpl 类中，消息的处理 逻辑是在 pullMessage 这个函数里的 PullCallBack 中。在 PullCallBack 函数里有个 switch 语句，根据从 Broker 返回的消息类型做相应的处理。
```java
// 下面代码来自:com.alibaba.rocketmq.client.impl.consumer.DefaultMQPushConsumerImpl#pullMessage
PullCallback pullCallback = new PullCallback() {
    public void onSuccess(PullResult pullResult) {
        if (pullResult != null) {
            pullResult = DefaultMQPushConsumerImpl.this.pullAPIWrapper.processPullResult(pullRequest.getMessageQueue(), pullResult, subscriptionData);
            switch(pullResult.getPullStatus()) {
            case FOUND:
                long prevRequestOffset = pullRequest.getNextOffset();
                pullRequest.setNextOffset(pullResult.getNextBeginOffset());
                long pullRT = System.currentTimeMillis() - beginTimestamp;
                DefaultMQPushConsumerImpl.this.getConsumerStatsManager().incPullRT(pullRequest.getConsumerGroup(), pullRequest.getMessageQueue().getTopic(), pullRT);
                long firstMsgOffset = 9223372036854775807L;
                if (pullResult.getMsgFoundList() != null && !pullResult.getMsgFoundList().isEmpty()) {
                    firstMsgOffset = ((MessageExt)pullResult.getMsgFoundList().get(0)).getQueueOffset();
                    DefaultMQPushConsumerImpl.this.getConsumerStatsManager().incPullTPS(pullRequest.getConsumerGroup(), pullRequest.getMessageQueue().getTopic(), (long)pullResult.getMsgFoundList().size());
                    boolean dispathToConsume = processQueue.putMessage(pullResult.getMsgFoundList());
                    DefaultMQPushConsumerImpl.this.consumeMessageService.submitConsumeRequest(pullResult.getMsgFoundList(), processQueue, pullRequest.getMessageQueue(), dispathToConsume);
                    if (DefaultMQPushConsumerImpl.this.defaultMQPushConsumer.getPullInterval() > 0L) {
                        DefaultMQPushConsumerImpl.this.executePullRequestLater(pullRequest, DefaultMQPushConsumerImpl.this.defaultMQPushConsumer.getPullInterval());
                    } else {
                        DefaultMQPushConsumerImpl.this.executePullRequestImmediately(pullRequest);
                    }
                } else {
                    DefaultMQPushConsumerImpl.this.executePullRequestImmediately(pullRequest);
                }

                if (pullResult.getNextBeginOffset() < prevRequestOffset || firstMsgOffset < prevRequestOffset) {
                    DefaultMQPushConsumerImpl.this.log.warn("[BUG] pull message result maybe data wrong, nextBeginOffset: {} firstMsgOffset: {} prevRequestOffset: {}", new Object[]{pullResult.getNextBeginOffset(), firstMsgOffset, prevRequestOffset});
                }
                break;
            case NO_NEW_MSG:
                pullRequest.setNextOffset(pullResult.getNextBeginOffset());
                DefaultMQPushConsumerImpl.this.correctTagsOffset(pullRequest);
                DefaultMQPushConsumerImpl.this.executePullRequestImmediately(pullRequest);
                break;
            case NO_MATCHED_MSG:
                pullRequest.setNextOffset(pullResult.getNextBeginOffset());
                DefaultMQPushConsumerImpl.this.correctTagsOffset(pullRequest);
                DefaultMQPushConsumerImpl.this.executePullRequestImmediately(pullRequest);
                break;
            case OFFSET_ILLEGAL:
                DefaultMQPushConsumerImpl.this.log.warn("the pull request offset illegal, {} {}", pullRequest.toString(), pullResult.toString());
                pullRequest.setNextOffset(pullResult.getNextBeginOffset());
                pullRequest.getProcessQueue().setDropped(true);
                DefaultMQPushConsumerImpl.this.executeTaskLater(new Runnable() {
                    public void run() {
                        try {
                            DefaultMQPushConsumerImpl.this.offsetStore.updateOffset(pullRequest.getMessageQueue(), pullRequest.getNextOffset(), false);
                            DefaultMQPushConsumerImpl.this.offsetStore.persist(pullRequest.getMessageQueue());
                            DefaultMQPushConsumerImpl.this.rebalanceImpl.removeProcessQueue(pullRequest.getMessageQueue());
                            DefaultMQPushConsumerImpl.this.log.warn("fix the pull request offset, {}", pullRequest);
                        } catch (Throwable var2) {
                            DefaultMQPushConsumerImpl.this.log.error("executeTaskLater Exception", var2);
                        }

                    }
                }, 10000L);
            }
        }

    }

    public void onException(Throwable e) {
        if (!pullRequest.getMessageQueue().getTopic().startsWith("%RETRY%")) {
            DefaultMQPushConsumerImpl.this.log.warn("execute the pull request exception", e);
        }

        DefaultMQPushConsumerImpl.this.executePullRequestLater(pullRequest, 3000L);
    }
};
```
### 长轮询
DefaultMQPushConsuer 中使用“ PullRequest” 通过“长轮询” 方式达到 Push 效果的方法， 长轮询方式既有 Pull 的优点，又兼具 Push 方式的实时性。“ 长 轮 询” 的主动权还是掌握在 Consumer 手中，Broker 即使有大量消息积压，也不会主动推送 给 Consumer。 长轮询方式的局限性，是在 HOLD 住Consumer 请求的时候需要占用资源，它适合用在消息队列这种客户端连接 数可控的场景中。
Push 方式是 Server 端接收到消息后， 主动把消息推送给 Client 端，实时性高。对于一个提供队列服务的 Server 来说，用 Push 方式主动推送有很多弊端：首先是加大 Server 端的工作量，进而影响 Server 的性能；其次，Client 的处理能力各不相同，Client 的状态不受 Server 控制， 如果 Client 不能及时处理 Server 推送过来的消息， 会造成各种潜在问题。
Pull 方式是Client 端循环地从 Server 端拉取 消息，主动权在 Client 手里， 自己拉取到一定量消息后， 处理妥当了再接着 取。 Pull 方式的问题是循环拉取消息的间隔不好设定，间隔太短就处在一个“ 忙 等” 的状态，浪费资源；每个 Pull 的时间间隔太长，Server 端有消息到来时，有可能没有被及时处理。
“长轮询”方式 通过 Client 端和 Server 端的配合，达到既拥有Pull 的优点，又能达到保证实时性的 目的。
```java
// 代码来自:com.alibaba.rocketmq.client.impl.consumer.PullAPIWrapper#pullKernelImpl
PullMessageRequestHeader requestHeader = new PullMessageRequestHeader();
requestHeader.setConsumerGroup(this.consumerGroup);
requestHeader.setTopic(mq.getTopic());
requestHeader.setQueueId(mq.getQueueId());
requestHeader.setQueueOffset(offset);
requestHeader.setMaxMsgNums(maxNums);
requestHeader.setSysFlag(sysFlagInner);
requestHeader.setCommitOffset(commitOffset);
requestHeader.setSuspendTimeoutMillis(brokerSuspendMaxTimeMillis);
requestHeader.setSubscription(subExpression);
requestHeader.setSubVersion(subVersion);
String brokerAddr = findBrokerResult.getBrokerAddr();
if (PullSysFlag.hasClassFilterFlag(sysFlagInner)) {
    brokerAddr = this.computPullFromWhichFilterServer(mq.getTopic(), brokerAddr);
}

PullResult pullResult = this.mQClientFactory.getMQClientAPIImpl().pullMessage(brokerAddr, requestHeader, timeoutMillis, communicationMode, pullCallback);
```
requestHeader. setSuspendTimeoutMillis(brokerSus- pendMaxTimeMillis)，作用是设置 Broker 最长阻塞时间， 默认设置是15 秒，注意是 Broker 在没有新消息的时候才阻塞，有消息会立刻返回。
“长轮询” 服务端代码片段
```c
// 代码来自:package org.apache.rocketmq.broker.longpolling
if (this.brokerController.getBrokerConfig().isLongPollingEnable()){
    this.waitForRunning( 5 * 1000); 
} else {
  this.waitForRunning(this.brokerController.getBrokerConfig().getShortPollingTimeMills()); 
}
long beginLockTimestamp = this.systemClock.now(); 
this.checkHoldRequest(); 
long costTime = this.systemClock.now() - beginLockTimestamp; 
if (costTime > 5 * 1000) { 
    Log. info("[ NOTIFYME] check hold request cost {} ms.", costTime); 
}
```
从 Broker 的源码中可以看出，服务端接到新消息请求后，如果队列里没有新消息，并不急于返回，通过一个循环不断查看状态，每次 waitForRunning 一段时间（默认是 5 秒），然后再 Check。 默认情况下当 Broker 一直 没有新消息，第三次 Check 的时候，等待时间超过 Request 里面的 Broker-SuspendMaxTimeMillis，就返回空 结果。在等待的过程中， Broker 收到了新的消息后会直接调用 notifyMessageArriving 函数返回请求结果。“ 长 轮 询” 的核心是，Broker 端 HOLD 住客户端过来的请求一小段时间，在这个时间内有新消息到达，就利用现有的连接立刻返回消息给 Consumer。
## DefaultMQPushConsumer 的流量控制
PushConsumer 有个线程 池，消息处理逻辑在各个线程里同时执行，这个线程池的定义如下:
```java
// 代码来着DefaultMQPushConsumer.defaultMQPushConsumerImpl.consumeMessageService.ConsumeMessageConcurrentlyService.consumeExecutor

public ConsumeMessageConcurrentlyService(DefaultMQPushConsumerImpl defaultMQPushConsumerImpl, MessageListenerConcurrently messageListener) {
    this.defaultMQPushConsumerImpl = defaultMQPushConsumerImpl;
    this.messageListener = messageListener;
    this.defaultMQPushConsumer = this.defaultMQPushConsumerImpl.getDefaultMQPushConsumer();
    this.consumerGroup = this.defaultMQPushConsumer.getConsumerGroup();
    this.consumeRequestQueue = new LinkedBlockingQueue();
    this.consumeExecutor = new ThreadPoolExecutor(this.defaultMQPushConsumer.getConsumeThreadMin(), this.defaultMQPushConsumer.getConsumeThreadMax(), 60000L, TimeUnit.MILLISECONDS, this.consumeRequestQueue, new ThreadFactoryImpl("ConsumeMessageThread_"));
    this.scheduledExecutorService = Executors.newSingleThreadScheduledExecutor(new ThreadFactoryImpl("ConsumeMessageScheduledThread_"));
    this.CleanExpireMsgExecutors = Executors.newSingleThreadScheduledExecutor(new ThreadFactoryImpl("CleanExpireMsgScheduledThread_"));
}
```
另外 PullRequest 中定义了 messageQueue, processQueue.
processQueue是一个快照类来, 在 PushConsumer 运行的时候， 每个 Message Queue 都会有个对应的 ProcessQueue 对象， 保存了这个 Message Queue 消息处理状态的快照。
ProcessQueue 对象里主要的内容是一个 TreeMap 和 一个读写锁。TreeMap 里以 Message Queue 的 Offset 作为 Key，以消息 内容的引用为 Value，保存了所有从 MessageQueue 获取到，但是还未被处理的消息；读写锁控制着多个线程对 TreeMap 对象的并发访问。
回到 pull 逻辑, PushConsumer 会判断获取但还未处理的消息个数、消息总大小、Offset 的跨度，任何一个值超过设定的大小就隔一段时间再拉取消息，从而达到流量控制的目的。此外 ProcessQueue 还可以辅助实现顺序消费的逻辑。代码如下:
```java
// 代码来自 : com.alibaba.rocketmq.client.impl.consumer.DefaultMQPushConsumerImpl#pullMessage

long size = processQueue.getMsgCount().get();
if (size > (long)this.defaultMQPushConsumer.getPullThresholdForQueue()) {
    this.executePullRequestLater(pullRequest, 50L);
    if (this.flowControlTimes1++ % 1000L == 0L) {
        this.log.warn("the consumer message buffer is full, so do flow control, minOffset={}, maxOffset={}, size={}, pullRequest={}, flowControlTimes={}", new Object[]{processQueue.getMsgTreeMap().firstKey(), processQueue.getMsgTreeMap().lastKey(), size, pullRequest, this.flowControlTimes1});
    }

} else {
    if (!this.consumeOrderly) {
        if (processQueue.getMaxSpan() > (long)this.defaultMQPushConsumer.getConsumeConcurrentlyMaxSpan()) {
            this.executePullRequestLater(pullRequest, 50L);
            if (this.flowControlTimes2++ % 1000L == 0L) {
                this.log.warn("the queue's messages, span too long, so do flow control, minOffset={}, maxOffset={}, maxSpan={}, pullRequest={}, flowControlTimes={}", new Object[]{processQueue.getMsgTreeMap().firstKey(), processQueue.getMsgTreeMap().lastKey(), processQueue.getMaxSpan(), pullRequest, this.flowControlTimes2});
            }

            return;
        }
    } else {
        if (!processQueue.isLocked()) {
            this.executePullRequestLater(pullRequest, 3000L);
            this.log.info("pull message later because not locked in broker, {}", pullRequest);
            return;
        }

        if (!pullRequest.isLockedFirst()) {
            long offset = this.rebalanceImpl.computePullFromWhere(pullRequest.getMessageQueue());
            boolean brokerBusy = offset < pullRequest.getNextOffset();
            this.log.info("the first time to pull message, so fix offset from broker. pullRequest: {} NewOffset: {} brokerBusy: {}", new Object[]{pullRequest, offset, brokerBusy});
            if (brokerBusy) {
                this.log.info("[NOTIFYME]the first time to pull message, but pull request offset larger than broker consume offset. pullRequest: {} NewOffset: {}", pullRequest, offset);
            }

            pullRequest.setLockedFirst(true);
            pullRequest.setNextOffset(offset);
        }
    }

    final SubscriptionData subscriptionData = (SubscriptionData)this.rebalanceImpl.getSubscriptionInner().get(pullRequest.getMessageQueue().getTopic());
    if (null == subscriptionData) {
        this.executePullRequestLater(pullRequest, 3000L);
        this.log.warn("find the consumer's subscription failed, {}", pullRequest);
    } else {
        final long beginTimestamp = System.currentTimeMillis();
        PullCallback pullCallback = new PullCallback() {
            public void onSuccess(PullResult pullResult) {
                if (pullResult != null) {
                    pullResult = DefaultMQPushConsumerImpl.this.pullAPIWrapper.processPullResult(pullRequest.getMessageQueue(), pullResult, subscriptionData);
                    switch(pullResult.getPullStatus()) {
                        case FOUND:
                            long prevRequestOffset = pullRequest.getNextOffset();
                            pullRequest.setNextOffset(pullResult.getNextBeginOffset());
                            long pullRT = System.currentTimeMillis() - beginTimestamp;
                            DefaultMQPushConsumerImpl.this.getConsumerStatsManager().incPullRT(pullRequest.getConsumerGroup(), pullRequest.getMessageQueue().getTopic(), pullRT);
                            long firstMsgOffset = 9223372036854775807L;
                            if (pullResult.getMsgFoundList() != null && !pullResult.getMsgFoundList().isEmpty()) {
                                firstMsgOffset = ((MessageExt)pullResult.getMsgFoundList().get(0)).getQueueOffset();
                                DefaultMQPushConsumerImpl.this.getConsumerStatsManager().incPullTPS(pullRequest.getConsumerGroup(), pullRequest.getMessageQueue().getTopic(), (long)pullResult.getMsgFoundList().size());
                                boolean dispathToConsume = processQueue.putMessage(pullResult.getMsgFoundList());
                                DefaultMQPushConsumerImpl.this.consumeMessageService.submitConsumeRequest(pullResult.getMsgFoundList(), processQueue, pullRequest.getMessageQueue(), dispathToConsume);
                                if (DefaultMQPushConsumerImpl.this.defaultMQPushConsumer.getPullInterval() > 0L) {
                                    DefaultMQPushConsumerImpl.this.executePullRequestLater(pullRequest, DefaultMQPushConsumerImpl.this.defaultMQPushConsumer.getPullInterval());
                                } else {
                                    DefaultMQPushConsumerImpl.this.executePullRequestImmediately(pullRequest);
                                }
                            } else {
                                DefaultMQPushConsumerImpl.this.executePullRequestImmediately(pullRequest);
                            }

                            if (pullResult.getNextBeginOffset() < prevRequestOffset || firstMsgOffset < prevRequestOffset) {
                                DefaultMQPushConsumerImpl.this.log.warn("[BUG] pull message result maybe data wrong, nextBeginOffset: {} firstMsgOffset: {} prevRequestOffset: {}", new Object[]{pullResult.getNextBeginOffset(), firstMsgOffset, prevRequestOffset});
                            }
                            break;
                        case NO_NEW_MSG:
                            pullRequest.setNextOffset(pullResult.getNextBeginOffset());
                            DefaultMQPushConsumerImpl.this.correctTagsOffset(pullRequest);
                            DefaultMQPushConsumerImpl.this.executePullRequestImmediately(pullRequest);
                            break;
                        case NO_MATCHED_MSG:
                            pullRequest.setNextOffset(pullResult.getNextBeginOffset());
                            DefaultMQPushConsumerImpl.this.correctTagsOffset(pullRequest);
                            DefaultMQPushConsumerImpl.this.executePullRequestImmediately(pullRequest);
                            break;
                        case OFFSET_ILLEGAL:
                            DefaultMQPushConsumerImpl.this.log.warn("the pull request offset illegal, {} {}", pullRequest.toString(), pullResult.toString());
                            pullRequest.setNextOffset(pullResult.getNextBeginOffset());
                            pullRequest.getProcessQueue().setDropped(true);
                            DefaultMQPushConsumerImpl.this.executeTaskLater(new Runnable() {
                                public void run() {
                                    try {
                                        DefaultMQPushConsumerImpl.this.offsetStore.updateOffset(pullRequest.getMessageQueue(), pullRequest.getNextOffset(), false);
                                        DefaultMQPushConsumerImpl.this.offsetStore.persist(pullRequest.getMessageQueue());
                                        DefaultMQPushConsumerImpl.this.rebalanceImpl.removeProcessQueue(pullRequest.getMessageQueue());
                                        DefaultMQPushConsumerImpl.this.log.warn("fix the pull request offset, {}", pullRequest);
                                    } catch (Throwable var2) {
                                        DefaultMQPushConsumerImpl.this.log.error("executeTaskLater Exception", var2);
                                    }

                                }
                            }, 10000L);
                    }
                }

            }

            public void onException(Throwable e) {
                if (!pullRequest.getMessageQueue().getTopic().startsWith("%RETRY%")) {
                    DefaultMQPushConsumerImpl.this.log.warn("execute the pull request exception", e);
                }

                DefaultMQPushConsumerImpl.this.executePullRequestLater(pullRequest, 3000L);
            }
        };
        boolean commitOffsetEnable = false;
        long commitOffsetValue = 0L;
        if (MessageModel.CLUSTERING == this.defaultMQPushConsumer.getMessageModel()) {
            commitOffsetValue = this.offsetStore.readOffset(pullRequest.getMessageQueue(), ReadOffsetType.READ_FROM_MEMORY);
            if (commitOffsetValue > 0L) {
                commitOffsetEnable = true;
            }
        }

        String subExpression = null;
        boolean classFilter = false;
        SubscriptionData sd = (SubscriptionData)this.rebalanceImpl.getSubscriptionInner().get(pullRequest.getMessageQueue().getTopic());
        if (sd != null) {
            if (this.defaultMQPushConsumer.isPostSubscriptionWhenPull() && !sd.isClassFilterMode()) {
                subExpression = sd.getSubString();
            }

            classFilter = sd.isClassFilterMode();
        }

        int sysFlag = PullSysFlag.buildSysFlag(commitOffsetEnable, true, subExpression != null, classFilter);

        try {
            this.pullAPIWrapper.pullKernelImpl(pullRequest.getMessageQueue(), subExpression, subscriptionData.getSubVersion(), pullRequest.getNextOffset(), this.defaultMQPushConsumer.getPullBatchSize(), sysFlag, commitOffsetValue, 15000L, 30000L, CommunicationMode.ASYNC, pullCallback);
        } catch (Exception var21) {
            this.log.error("pullKernelImpl exception", var21);
            this.executePullRequestLater(pullRequest, 3000L);
        }

    }
}
```
