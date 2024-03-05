# RocketMQ 队列读取位置信息的存储

> 《RocketMQ实战与原理解析》

## OffsetStore
实际运行中的系统，难免会遇到重新消费某条消息、跳过一段时间内的消息等情况。这些异常情况的处理，都和 Offset 有关。
本节主要分析 Offset 的存储位置，以及如何根据需要调整 Offset 的值。首先来明确一下 Offset 的含义，RocketMQ 中，一种类型的消息会放到一个 Topic 里，为了能够并行，一般一个 Topic 会有多个 Message Queue（也可以设置成一个），Offset 是指某个 Topic 下 的一条消息在某个 Message Queue 里的位置，通过 Offset 的值可以定位到这条消息，或者指示 Consumer 从这条消息开始向后继续处理。
如图所示是 Offset 的类结构，主要分为本地文件类型和 Broker 代存的类型两种。
## DefaultMQPushConsumer
对于 DefaultMQPushConsumer 来说，默认是 CLUSTERING 模式，也就是同一个 Consumer group 里的多个消费者每人消费一部分，各自收到的消息内容不一样。
这种情况下，由 Broker 端存储和控制 Offset 的值，使用 RemoteBrokerOffsetStore 结构。
在 DefaultMQPushConsumer 里的 BROADCASTING 模式下，每个 Consumer 都收到这个 Topic 的全部消息，各个 Consumer 间相互没有干扰，
RocketMQ 使用 LocalFileOffsetStore，把 Offset 存到本地。 OffsetStore 使用 Json 格式存储，简洁明了，下面是个例子：
```json
{"OffsetTable":{{" brokerName":" localhost", "QueueId": 1," Topic":" broker1" }: 1,{ "brokerName":" localhost", "QueueId": 2," Topic":" broker1" }:2, { "brokerName":" localhost", "QueueId": 0, "Topic":" broker1" }:3 } }
```
## DefaultMQPullConsumer
在使用 DefaultMQPushConsumer 的时候，我们不用关心 OffsetStore 的事，但是如果 PullConsumer，我们就要自己处理 OffsetStore 了。
```java
@Slf4j
public class LocalOffsetStoreExt implements OffsetStore {

    private final String groupName;

    private final String storePath;

    private final ConcurrentMap<MessageQueue, AtomicLong> offsetTable = new ConcurrentHashMap<>();

    public LocalOffsetStoreExt(String storePath, String groupName) {
        this.groupName = groupName;
        this.storePath = storePath;
    }

    @Override
    public void load() {
        OffsetSerializeWrapper offsetSerializeWrapper = this.readLocalOffset();
        if (offsetSerializeWrapper != null && offsetSerializeWrapper.getOffsetTable() != null) {
            offsetTable.putAll(offsetSerializeWrapper.getOffsetTable());
            for (MessageQueue mq : offsetSerializeWrapper.getOffsetTable().keySet()) {
                AtomicLong offset = offsetSerializeWrapper.getOffsetTable().get(mq);
                log.info("load Consumer' s Offset, {} {} {} ", this.groupName, mq, offset.get());
            }
        }
    }


    @Override
    public void updateOffset(MessageQueue mq, long offset, boolean increaseOnly) {
        if (mq != null) {
            AtomicLong offsetOld = this.offsetTable.get(mq);
            if (null == offsetOld) {
                this.offsetTable.putIfAbsent(mq, new AtomicLong(offset));
            } else {
                offsetOld.set(offset);
            }
            if (null != offsetOld) {
                if (increaseOnly) {
                    MixAll.compareAndIncreaseOnly(offsetOld, offset);
                } else {
                    offsetOld.set(offset);
                }
            }
        }
    }


    @Override
    public long readOffset(MessageQueue mq, ReadOffsetType type) {
        if (mq != null) {
            AtomicLong offset = this.offsetTable.get(mq);
            if (offset != null) {
                return offset.get();
            }
        }
        return 0;
    }

    @Override
    public void persistAll(Set<MessageQueue> mqs) {
        if (null == mqs || mqs.isEmpty()) {
            return;
        }
        OffsetSerializeWrapper offsetSerializeWrapper = new OffsetSerializeWrapper();
        for (Map.Entry<MessageQueue, AtomicLong> entry : this.offsetTable.entrySet()) {
            if (mqs.contains(entry.getKey())) {
                AtomicLong offset = entry.getValue();
                offsetSerializeWrapper.getOffsetTable().put(entry.getKey(), offset);
            }
        }
        String jsonString = offsetSerializeWrapper.toJson(true);
        if (jsonString != null) {
            try {
                MixAll.string2File(jsonString, this.storePath);
            } catch (IOException e) {
                e.printStackTrace();

            }
        }
    }


    @Override
    public void persist(MessageQueue mq) {
        log.info("mq = {}", mq);
    }

    @Override
    public void removeOffset(MessageQueue mq) {
    }


    @Override
    public Map<MessageQueue, Long> cloneOffsetTable(String topic) {
        Map<MessageQueue, Long> cloneOffsetTable = new HashMap<>();
        Iterator<Map.Entry<MessageQueue, AtomicLong>> iterator = this.offsetTable.entrySet().iterator();

        while (true) {
            Map.Entry<MessageQueue, AtomicLong> entry;
            MessageQueue mq;
            do {
                if (!iterator.hasNext()) {
                    return cloneOffsetTable;
                }
                entry = iterator.next();
                mq = entry.getKey();
            } while (!UtilAll.isBlank(topic) && !topic.equals(mq.getTopic()));

            cloneOffsetTable.put(mq, entry.getValue().get());
        }
    }

    private OffsetSerializeWrapper readLocalOffset() {
        String content;
        content = MixAll.file2String(this.storePath);
        if (null == content || content.length() == 0) {
            return null;
        } else {
            OffsetSerializeWrapper offsetSerializeWrapper = null;
            try {
                offsetSerializeWrapper = OffsetSerializeWrapper.fromJson(content, OffsetSerializeWrapper.class);
            } catch (Exception e) {
                e.printStackTrace();
            }
            return offsetSerializeWrapper;
        }
    }
}
```
LocalFileOffsetStore 代码里把 Offset 存到了内存，没有持久化存储，这样就可能因为程序的异常或重启而丢失 Offset，在实际应用中不推荐这样做。
了解 OffsetStore 的存储机制以后，我们看看如何设置 Consumer 读取消息的初始位置。
DefaultMQPushConsumer 类里有个函数用来设置从哪儿开始消费消息：比如setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET)，这个语句设置从最小的 Offset 开始读取。如果从队列开始到感兴趣的消息之间有很大的范围，用 CONSUME_FROM_FIRST_OFFSET 参数就不合适了，可以设置从某个时间开始消费消息，比如 Consumer.setConsumeTimestamp("20131223171201")，时间戳格式是精确到秒的。
注意设置读取位置不是每次都有效，它的优先级默认在 OffsetStore 后面，比如在 DefaultMQPushConsumer 的 BROADCASTING 方式下，默认是从 Broker 里读取某个 Topic 对应 ConsumerGroup 的 Offset， 当读取不到 Offset 的时候，ConsumeFromWhere 的设置才生效。大部分情况下这个设置在 Consumer Group 初次启动时有效。如果 Consumer 正常运行后被停止，然后再启动，会接着上次的 Offset 开始消费，ConsumeFromWhere 的设置无效。
