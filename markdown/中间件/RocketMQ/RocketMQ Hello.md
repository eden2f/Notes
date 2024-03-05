# RocketMQ Hello

> 《RocketMQ实战与原理解析》

## Maven依赖
```xml
<dependency>
	<groupId>com.alibaba.rocketmq</groupId>
	<artifactId>rocketmq-client</artifactId>
	<version>3.6.2.Final</version>
</dependency>
```
## 发送消息
```java
public class SyncProducer {

    public static void main(String[] args) throws Exception {
        //Instantiate with a Producer group name.
        DefaultMQProducer producer = new DefaultMQProducer("please_rename_unique_group_name");
        producer.setNamesrvAddr("127.0.0.1:9876");
        //Launch the instance.
        producer.start();
        for (int i = 0; i < 100; i++) {
            //Create a Message instance, specifying Topic, tag and Message body.
            Message msg = new Message("TopicTest", "TagA",
                    ("Hello RocketMQ " + i).getBytes(RemotingHelper.DEFAULT_CHARSET));
            //Call send Message to deliver Message to one of brokers.
            SendResult sendResult = producer.send(msg);
            System.out.println(String.format("%s", sendResult));
        }
        //Shut down once the Producer instance is not longer in use.
        producer.shutdown();
    }

}
```
## 接收消息
```java
public class Consumer {

    public static void main(String[] args) throws MQClientException {
        /* Instantiate with specified Consumer group name. */
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("please_rename_unique_group_name");
        /* Specify name server addresses. */
        consumer.setNamesrvAddr("127.0.0.1:9876");
        /* Specify where to start in case the specified Consumer group is a brand new one. */
        consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);
        consumer.setMessageModel(MessageModel.BROADCASTING);
        /* Subscribe one more more Topics to consume. */
        consumer.subscribe("TopicTest", "*");
        /* * Register callback to execute on arrival of Messages fetched from brokers. */
        consumer.registerMessageListener(
                new MessageListenerConcurrently() {
                    public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs, ConsumeConcurrentlyContext context) {
                        System.out.printf(Thread.currentThread().getName() + " Receive New Messages: " + msgs + "%n");
                        System.out.println();
                        return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
                    }
                });
        /* * Launch the Consumer instance.*/
        consumer.start();
    }

}
```
