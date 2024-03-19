# SpringStateMachine 用状态机控制业务状态扭转

> [转载原文 : https://www.codetd.com/article/1010726](https://www.codetd.com/article/1010726)
[Spring StateMachine 官方文档](https://docs.spring.io/autorepo/docs/spring-statemachine/2.2.0.RELEASE/reference)

# 一、状态机
有限状态机是一种用来进行对象行为建模的工具，其作用主要是描述对象在它的生命周期内所经历的状态序列，以及如何响应来自外界的各种事件。在电商场景（订单、物流、售后）、社交（IM消息投递）、分布式集群管理（分布式计算平台任务编排）等场景都有大规模的使用。
## 状态机的要素：
状态机可归纳为4个要素，现态、条件、动作、次态。“现态”和“条件”是因，“动作”和“次态”是果。
1 现态：指当前所处的状态
2 条件：又称“事件”，当一个条件被满足，将会触发一个动作，或者执行一次状态的迁移
3 动作：条件满足后执行的动作。动作执行完毕后，可以迁移到新的状态，也可以仍旧保持原状态。动作不是必须的，当条件满足后，也可以不执行任何动作，直接迁移到新的状态。
4 次态：条件满足后要迁往的新状态。“次态”是相对于“现态”而言的，“次态”一旦被激活，就转换成“现态”。
## 状态机动作类型：
进入动作：在进入状态时进行
退出动作：在退出状态时进行
输入动作：依赖于当前状态和输入条件进行
转移动作：在进行特定转移时进行
# 二、spring statemachine
spring statemachine是使用 Spring框架下的状态机概念创建的一种应用程序开发框架。它使得状态机结构层次化，简化了配置状态机的过程。
## 例子一：简单订单流程
![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240319232025.png#id=PLqwk&originHeight=500&originWidth=1562&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
### 1 引入依赖
```xml
<dependency>
	<groupId>org.springframework.statemachine</groupId>
	<artifactId>spring-statemachine-core</artifactId>
	<version>2.2.0.RELEASE</version>
</dependency>
```
### 2 创建订单状态枚举类和状态转换枚举类
```java
/**
 * 订单状态
 */
public enum OrderStatus {
    // 待支付，待发货，待收货，订单结束
    WAIT_PAYMENT, WAIT_DELIVER, WAIT_RECEIVE, FINISH;
}
/**
 * 订单状态改变事件
 */
public enum OrderStatusChangeEvent {
    // 支付，发货，确认收货
    PAYED, DELIVERY, RECEIVED;
}
```
### 3 添加配置
```java
/**
 * 订单状态机配置
 */
@Configuration
@EnableStateMachine(name = "orderStateMachine")
public class OrderStateMachineConfig extends StateMachineConfigurerAdapter<OrderStatus, OrderStatusChangeEvent> {

    /**
     * 配置状态
     * @param states
     * @throws Exception
     */
    @Override
    public void configure(StateMachineStateConfigurer<OrderStatus, OrderStatusChangeEvent> states) throws Exception {
        states
                .withStates()
                .initial(OrderStatus.WAIT_PAYMENT)
                .states(EnumSet.allOf(OrderStatus.class));
    }

    /**
     * 配置状态转换事件关系
     * @param transitions
     * @throws Exception
     */
    @Override
    public void configure(StateMachineTransitionConfigurer<OrderStatus, OrderStatusChangeEvent> transitions) throws Exception {
        transitions
                .withExternal().source(OrderStatus.WAIT_PAYMENT).target(OrderStatus.WAIT_DELIVER).event(OrderStatusChangeEvent.PAYED)
                .and()
                .withExternal().source(OrderStatus.WAIT_DELIVER).target(OrderStatus.WAIT_RECEIVE).event(OrderStatusChangeEvent.DELIVERY)
                .and()
                .withExternal().source(OrderStatus.WAIT_RECEIVE).target(OrderStatus.FINISH).event(OrderStatusChangeEvent.RECEIVED);
    }

    /**
     * 持久化配置
     * 实际使用中，可以配合redis等，进行持久化操作
     * @return
     */
    @Bean
    public StateMachinePersister<OrderStatus, OrderStatusChangeEvent, Order> persister(){
        return new DefaultStateMachinePersister<>(new StateMachinePersist<OrderStatus, OrderStatusChangeEvent, Order>() {
            @Override
            public void write(StateMachineContext<OrderStatus, OrderStatusChangeEvent> context, Order order) throws Exception {
                //此处并没有进行持久化操作
                System.out.println("持久化订单信息, order = " + order);
            }

            @Override
            public StateMachineContext<OrderStatus, OrderStatusChangeEvent> read(Order order) throws Exception {
                //此处直接获取order中的状态，其实并没有进行持久化读取操作
                return new DefaultStateMachineContext<>(order.getStatus(), null, null, null);
            }
        });
    }
}
```
### 4 添加订单状态监听器
```java
@Component("orderStateListener")
@WithStateMachine(name = "orderStateMachine")
public class OrderStateListenerImpl {

    @OnTransition(source = "WAIT_PAYMENT", target = "WAIT_DELIVER")
    public boolean payTransition(Message<OrderStatusChangeEvent> message) {
        Order order = (Order) message.getHeaders().get("order");
        Long payOrder = (Long) message.getHeaders().get("payOrder");
        Assert.isTrue(!StringUtils.isEmpty(payOrder), "payOrder不能为空");
        order.setPayOrder(payOrder);
        order.setStatus(OrderStatus.WAIT_DELIVER);
        System.out.println("支付 headers=" + message.getHeaders().toString());
        return true;
    }

    @OnTransition(source = "WAIT_DELIVER", target = "WAIT_RECEIVE")
    public boolean deliverTransition(Message<OrderStatusChangeEvent> message) {
        Order order = (Order) message.getHeaders().get("order");
        Long deliverOrder = (Long) message.getHeaders().get("deliverOrder");
        Assert.isTrue(!StringUtils.isEmpty(deliverOrder), "deliverOrder不能为空");
        order.setDeliverOrder(deliverOrder);
        order.setStatus(OrderStatus.WAIT_RECEIVE);
        System.out.println("发货 headers=" + message.getHeaders().toString());
        return true;
    }

    @OnTransition(source = "WAIT_RECEIVE", target = "FINISH")
    public boolean receiveTransition(Message<OrderStatusChangeEvent> message) {
        Order order = (Order) message.getHeaders().get("order");
        Long confirmReceipt = (Long) message.getHeaders().get("confirmReceipt");
        Assert.isTrue(!StringUtils.isEmpty(confirmReceipt), "confirmReceipt不能为空");
        order.setConfirmReceipt(confirmReceipt);
        order.setStatus(OrderStatus.FINISH);
        System.out.println("收货 headers=" + message.getHeaders().toString());
        return true;
    }
}
```
### 5 service中使用
```java
@Service("orderService")
public class OrderServiceImpl implements OrderService {

    @Resource
    private StateMachine<OrderStatus, OrderStatusChangeEvent> orderStateMachine;

    @Resource
    private StateMachinePersister<OrderStatus, OrderStatusChangeEvent, Order> persister;

    private Long id = 1L;

    private final Map<Long, Order> orders = new HashMap<>();

    @Override
    public Order creat(Long skuId, Integer skuNum) {
        System.out.println(String.format("threadName=%s, 创建订单, skuId = %s, skuNum = %s", Thread.currentThread().getName(), skuId, skuNum));
        Order order = new Order();
        order.setSkuId(skuId);
        order.setSkuNum(skuNum);
        order.setStatus(OrderStatus.WAIT_PAYMENT);
        order.setId(id++);
        orders.put(order.getId(), order);
        return order;
    }

    @Override
    public Order pay(Long id, Long payOrder) {
        Order order = orders.get(id);
        System.out.println(String.format("threadName=%s, 尝试支付, id = %s, payOrder = %s", Thread.currentThread().getName(), id, payOrder));
        Message<OrderStatusChangeEvent> message = MessageBuilder.withPayload(OrderStatusChangeEvent.PAYED)
                .setHeader("order", order)
                .setHeader("payOrder", payOrder)
                .build();
        if (!sendEvent(message, order)) {
            System.out.println(String.format("threadName=%s, 支付失败，状态异常, id = %s, payOrder = %s", Thread.currentThread().getName(), id, payOrder));
        }
        return orders.get(id);
    }


    @Override
    public Order deliver(Long id, Long deliverOrder) {
        Order order = orders.get(id);
        System.out.println(String.format("threadName=%s, 尝试发货, id = %s, deliverOrder = %s", Thread.currentThread().getName(), id, deliverOrder));
        Message<OrderStatusChangeEvent> message = MessageBuilder.withPayload(OrderStatusChangeEvent.DELIVERY)
                .setHeader("order", order)
                .setHeader("deliverOrder", deliverOrder)
                .build();
        if (!sendEvent(message, orders.get(id))) {
            System.out.println(String.format("threadName=%s, 发货失败，状态异常, id = %s, deliverOrder = %s", Thread.currentThread().getName(), id, deliverOrder));
        }
        return orders.get(id);
    }


    @Override
    public Order receive(Long id, Long confirmReceipt) {
        Order order = orders.get(id);
        System.out.println(String.format("threadName=%s, 尝试收货, id = %s, confirmReceipt = %s", Thread.currentThread().getName(), id, confirmReceipt));
        Message<OrderStatusChangeEvent> message = MessageBuilder.withPayload(OrderStatusChangeEvent.RECEIVED)
                .setHeader("order", order)
                .setHeader("confirmReceipt", confirmReceipt)
                .build();
        if (!sendEvent(message, orders.get(id))) {
            System.out.println(String.format("threadName=%s, 收货失败，状态异常, id = %s, confirmReceipt = %s", Thread.currentThread().getName(), id, confirmReceipt));
        }
        return orders.get(id);
    }


    @Override
    public Map<Long, Order> getOrders() {
        return orders;
    }


    /**
     * 发送订单状态转换事件
     *
     * @param message
     * @param order
     * @return
     */
    private synchronized boolean sendEvent(Message<OrderStatusChangeEvent> message, Order order) {
        boolean result = false;
        try {
            orderStateMachine.start();
            //尝试恢复状态机状态
            persister.restore(orderStateMachine, order);
            //添加延迟用于线程安全测试
            Thread.sleep(1000);
            result = orderStateMachine.sendEvent(message);
            //持久化状态机状态
            persister.persist(orderStateMachine, order);
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            orderStateMachine.stop();
        }
        return result;
    }
}
```
### 6 测试
```java
@SpringBootTest
public class OrderServiceImplTest {

    @Autowired
    private OrderService orderService;

    @Test
    public void testMultThread() throws InterruptedException {
        Random randon = new Random();
        Order order1 = orderService.creat((long) randon.nextInt(999), randon.nextInt(999));
        Order order2 = orderService.creat((long) randon.nextInt(999), randon.nextInt(999));

        orderService.pay(order1.getId(), (long) randon.nextInt(999));

        new Thread(() -> {
            orderService.deliver(order1.getId(), (long) randon.nextInt(999));
            orderService.receive(order1.getId(), (long) randon.nextInt(999));
        }).start();

        orderService.pay(order2.getId(), (long) randon.nextInt(999));
        orderService.deliver(order2.getId(), (long) randon.nextInt(999));
        orderService.receive(order2.getId(), (long) randon.nextInt(999));

        Thread.currentThread().join(6 * 1000);

        System.out.println(orderService.getOrders());

    }

}
```
## 例子二：状态机工厂
有些时候，一个状态机不够用，因为我们可能要处理多个订单。这个时候就要用到了状态机工厂。
1、不同线程启用不同statemachine实例处理
2、用工厂模式创建statemachine，且用StateMachinePersist根据recruit对象不同状态反序列化statemachine
### 1 配置修改
```java
/**
 * 订单状态机配置
 */
@Configuration
@EnableStateMachineFactory(name = "orderStateMachineFactory")
public class OrderStateMachineConfig extends EnumStateMachineConfigurerAdapter<OrderStatus, OrderStatusChangeEvent> {

    /**
     * 订单状态机ID
     */
    public static final String orderStateMachineId = "orderStateMachineId";

    /**
     * 配置状态
     *
     * @param states
     * @throws Exception
     */
    @Override
    public void configure(StateMachineStateConfigurer<OrderStatus, OrderStatusChangeEvent> states) throws Exception {
        states
                .withStates()
                .initial(OrderStatus.WAIT_PAYMENT)
                .states(EnumSet.allOf(OrderStatus.class));
    }

    /**
     * 配置状态转换事件关系
     *
     * @param transitions
     * @throws Exception
     */
    @Override
    public void configure(StateMachineTransitionConfigurer<OrderStatus, OrderStatusChangeEvent> transitions) throws Exception {
        transitions
                .withExternal().source(OrderStatus.WAIT_PAYMENT).target(OrderStatus.WAIT_DELIVER).event(OrderStatusChangeEvent.PAYED)
                .and()
                .withExternal().source(OrderStatus.WAIT_DELIVER).target(OrderStatus.WAIT_RECEIVE).event(OrderStatusChangeEvent.DELIVERY)
                .and()
                .withExternal().source(OrderStatus.WAIT_RECEIVE).target(OrderStatus.FINISH).event(OrderStatusChangeEvent.RECEIVED);
    }

    /**
     * 持久化配置
     * 实际使用中，可以配合redis等，进行持久化操作
     *
     * @return
     */
    @Bean
    public StateMachinePersister<OrderStatus, OrderStatusChangeEvent, Order> persister() {
        return new DefaultStateMachinePersister<>(new StateMachinePersist<OrderStatus, OrderStatusChangeEvent, Order>() {
            @Override
            public void write(StateMachineContext<OrderStatus, OrderStatusChangeEvent> context, Order order) throws Exception {
                //此处并没有进行持久化操作
                order.setStatus(context.getState());
                System.out.println("持久化订单信息, order = " + order);
            }

            @Override
            public StateMachineContext<OrderStatus, OrderStatusChangeEvent> read(Order order) throws Exception {
                //此处直接获取order中的状态，其实并没有进行持久化读取操作
                StateMachineContext<OrderStatus, OrderStatusChangeEvent> result = new DefaultStateMachineContext<>(order.getStatus(), null, null, null, null, orderStateMachineId);
                return result;
            }
        });
    }
}
```
### 2 service中使用
```java
@Service("orderService")
public class OrderServiceImpl implements OrderService {

    @Resource
    private StateMachineFactory<OrderStatus, OrderStatusChangeEvent> orderStateMachineFactory;

    @Resource
    private StateMachinePersister<OrderStatus, OrderStatusChangeEvent, Order> persister;

    private Long id = 1L;

    private final Map<Long, Order> orders = new HashMap<>();

    @Override
    public Order creat(Long skuId, Integer skuNum) {
        System.out.println(String.format("threadName=%s, 创建订单, skuId = %s, skuNum = %s", Thread.currentThread().getName(), skuId, skuNum));
        Order order = new Order();
        order.setSkuId(skuId);
        order.setSkuNum(skuNum);
        order.setStatus(OrderStatus.WAIT_PAYMENT);
        order.setId(id++);
        orders.put(order.getId(), order);
        return order;
    }

    @Override
    public Order pay(Long id, Long payOrder) {
        Order order = orders.get(id);
        System.out.println(String.format("threadName=%s, 尝试支付, id = %s, payOrder = %s", Thread.currentThread().getName(), id, payOrder));
        Message<OrderStatusChangeEvent> message = MessageBuilder.withPayload(OrderStatusChangeEvent.PAYED)
                .setHeader("order", order)
                .setHeader("payOrder", payOrder)
                .build();
        if (!sendEvent(message, order)) {
            System.out.println(String.format("threadName=%s, 支付失败，状态异常, id = %s, payOrder = %s", Thread.currentThread().getName(), id, payOrder));
        }
        return orders.get(id);
    }


    @Override
    public Order deliver(Long id, Long deliverOrder) {
        Order order = orders.get(id);
        System.out.println(String.format("threadName=%s, 尝试发货, id = %s, deliverOrder = %s", Thread.currentThread().getName(), id, deliverOrder));
        Message<OrderStatusChangeEvent> message = MessageBuilder.withPayload(OrderStatusChangeEvent.DELIVERY)
                .setHeader("order", order)
                .setHeader("deliverOrder", deliverOrder)
                .build();
        if (!sendEvent(message, orders.get(id))) {
            System.out.println(String.format("threadName=%s, 发货失败，状态异常, id = %s, deliverOrder = %s", Thread.currentThread().getName(), id, deliverOrder));
        }
        return orders.get(id);
    }


    @Override
    public Order receive(Long id, Long confirmReceipt) {
        Order order = orders.get(id);
        System.out.println(String.format("threadName=%s, 尝试收货, id = %s, confirmReceipt = %s", Thread.currentThread().getName(), id, confirmReceipt));
        Message<OrderStatusChangeEvent> message = MessageBuilder.withPayload(OrderStatusChangeEvent.RECEIVED)
                .setHeader("order", order)
                .setHeader("confirmReceipt", confirmReceipt)
                .build();
        if (!sendEvent(message, orders.get(id))) {
            System.out.println(String.format("threadName=%s, 收货失败，状态异常, id = %s, confirmReceipt = %s", Thread.currentThread().getName(), id, confirmReceipt));
        }
        return orders.get(id);
    }


    @Override
    public Map<Long, Order> getOrders() {
        return orders;
    }


    /**
     * 发送订单状态转换事件
     *
     * @param message
     * @param order
     * @return
     */
    private boolean sendEvent(Message<OrderStatusChangeEvent> message, Order order) {
        synchronized (String.valueOf(order.getId()).intern()) {
            boolean result = false;
            StateMachine<OrderStatus, OrderStatusChangeEvent> orderStateMachine = orderStateMachineFactory.getStateMachine(OrderStateMachineConfig.orderStateMachineId);
            System.out.println("id=" + order.getId() + " 状态机 orderStateMachine" + orderStateMachine);
            try {
                orderStateMachine.start();
                //尝试恢复状态机状态
                persister.restore(orderStateMachine, order);
                System.out.println("id=" + order.getId() + " 状态机 orderStateMachine id=" + orderStateMachine.getId());
                //添加延迟用于线程安全测试
                Thread.sleep(1000);
                result = orderStateMachine.sendEvent(message);
                //持久化状态机状态
                persister.persist(orderStateMachine, order);
            } catch (Exception e) {
                e.printStackTrace();
            } finally {
                orderStateMachine.stop();
            }
            return result;
        }
    }
}
```
### 3 listener中配置id
```java
@Component("orderStateListener")
@WithStateMachine(id = OrderStateMachineConfig.orderStateMachineId)
public class OrderStateListenerImpl {

    @OnTransition(source = "WAIT_PAYMENT", target = "WAIT_DELIVER")
    public boolean payTransition(Message<OrderStatusChangeEvent> message) {
        System.out.println("----------------------------");
        Order order = (Order) message.getHeaders().get("order");
        order.setStatus(OrderStatus.WAIT_DELIVER);
        System.out.println("支付 headers=" + message.getHeaders().toString() + " event=" + message.getPayload());
        System.out.println("----------------------------");
        return true;
    }

    @OnTransition(source = "WAIT_DELIVER", target = "WAIT_RECEIVE")
    public boolean deliverTransition(Message<OrderStatusChangeEvent> message) {
        System.out.println("----------------------------");
        Order order = (Order) message.getHeaders().get("order");
        order.setStatus(OrderStatus.WAIT_RECEIVE);
        System.out.println("发货 headers=" + message.getHeaders().toString() + " event=" + message.getPayload());
        System.out.println("----------------------------");
        return true;
    }

    @OnTransition(source = "WAIT_RECEIVE", target = "FINISH")
    public boolean receiveTransition(Message<OrderStatusChangeEvent> message) {
        System.out.println("----------------------------");
        Order order = (Order) message.getHeaders().get("order");
        order.setStatus(OrderStatus.FINISH);
        System.out.println("收货 headers=" + message.getHeaders().toString() + " event=" + message.getPayload());
        System.out.println("----------------------------");
        return true;
    }
}
```
