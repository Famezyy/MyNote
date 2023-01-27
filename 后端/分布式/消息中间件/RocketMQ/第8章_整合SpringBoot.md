# 第8章_整合SpringBoot

## 1.快速入门

### 1.1 依赖导入

```xml
<dependency>
    <groupId>org.apache.rocketmq</groupId>
    <artifactId>rocketmq-spring-boot-starter</artifactId>
    <version>2.2.0</version>
</dependency>
```

### 1.2 配置文件

```properties
rocketmq.name-server=192.168.11.101:9876
rocketmq.producer.group=rocketMQ_producer
# rocketmq.consumer.group=rocketMQ_consumer
```

### 1.3 producer

```java
@Service
public class ProducerService {

    @Autowired
    RocketMQTemplate rocketMQTemplate;

    public void sendMessage() {
        rocketMQTemplate.convertAndSend("SimpleTopic","hello spring rocket");
    }
}
```

### 1.4 consumer

```java
@Service
@RocketMQMessageListener(topic = "SimpleTopic", consumerGroup = "${rocketmq.consumer.group}")
public class ConsumerService implements RocketMQListener<String> {
    @Override
    public void onMessage(String o) {
        System.out.println("received message:" + o);
    }
}
```

## 2.各种消息场景

### 2.1 发送同步消息

发送同步消息指 producer 向 broker 发送消息，执行 API 时同步等待，直到 broker 服务器返回发送结果。

```java
@Service
public class ProducerService {

    @Autowired
    RocketMQTemplate rocketMQTemplate;

    public void sendMessage() {
        SendResult sendResult = rocketMQTemplate.syncSend("SimpleTopic", "hello spring rocket");
        System.out.println(sendResult.getSendStatus());
    }
}
```

### 2.2 发送异步消息

```java
public void sendMessage() throws InterruptedException {
    rocketMQTemplate.asyncSend("SimpleTopic", "hello spring rocket", new SendCallback() {
        @Override
        public void onSuccess(SendResult sendResult) {
            System.out.println(sendResult.getSendStatus());
        }
        @Override
        public void onException(Throwable e) {
            System.out.println("send failed");
        }
    });
    Thread.sleep(10000);
}
```

### 2.3 发送单向消息

不需要等待 broker 返回消息发送的结果。

```java
public void sendMessage() throws InterruptedException {
    rocketMQTemplate.sendOneWay("SimpleTopic", "hello spring rocket");
    Thread.sleep(10000);
}
```

### 2.4 消费者消费模式

在消费者中通过`RocketMQMessageListener`的`messageModel`属性设置，默认集群模式。

```java
@RocketMQMessageListener(topic = "SimpleTopic", consumerGroup = "${rocketmq.consumer.group}", messageModel = MessageModel.BROADCASTING)
```

### 2.5 发送顺序消息

**生产者**

使用`sendOrderly`方法，通过第三个参数`hashKey`绑定单独的队列，其他方法也有`hashKey`参数的重载。

```java
public void sendMessage() throws InterruptedException {
    rocketMQTemplate.syncSendOrderly("SimpleTopic", "1,create", "1");
    rocketMQTemplate.syncSendOrderly("SimpleTopic", "2,create", "2");
    rocketMQTemplate.syncSendOrderly("SimpleTopic", "1,complete", "1");
    rocketMQTemplate.syncSendOrderly("SimpleTopic", "3,create", "3");
    rocketMQTemplate.syncSendOrderly("SimpleTopic", "2,complete", "2");
    rocketMQTemplate.syncSendOrderly("SimpleTopic", "3,complete", "3");
    Thread.sleep(10000);
}
```

**消费者**

修改`RocketMQMessageListener`的参数`consumeMode`为`ORDERLY`。

```java
@RocketMQMessageListener(topic = "SimpleTopic", consumerGroup = "${rocketmq.consumer.group}", consumeMode = ConsumeMode.ORDERLY)
```

**结果分析**

可以发现每个编号的消息按顺序执行。

```bash
received message:1,create
received message:2,create
received message:3,create
received message:3,complete
received message:1,complete
received message:2,complete
```

### 2.6 延迟消息

每个带 delayLevel 参数的方法，同时拥有 timeout 参数，即消息发送超时时间，默认 3000 毫秒。

```java
public void sendMessage() throws InterruptedException {
	// String destination, Message<?> message, long timeout, int delayLevel
    rocketMQTemplate.syncSend("SimpleTopic", MessageBuilder.withPayload("create delay message").build(), 3000, 2);
    Thread.sleep(20000);
}
```

### 2.7 事务消息

只需修改生产者：

```java
@Service
public class ProducerService {

    @Autowired
    RocketMQTemplate rocketMQTemplate;

    public void sendMessage() throws InterruptedException {
        // 调用 sendMessageInTransaction 方法
        rocketMQTemplate.sendMessageInTransaction("SimpleTopic", MessageBuilder.withPayload("hello transaction").build(), null);
        Thread.sleep(20000);
    }
}

// 使用 @RocketMQTransactionListener 注解
@RocketMQTransactionListener
class TransactionListenerImpl implements RocketMQLocalTransactionListener {

    @Override
    public RocketMQLocalTransactionState executeLocalTransaction(Message msg, Object arg) {
        System.out.println("wait a while");
        return RocketMQLocalTransactionState.UNKNOWN;
    }

    @Override
    public RocketMQLocalTransactionState checkLocalTransaction(Message msg) {
        System.out.println("success");
        return RocketMQLocalTransactionState.COMMIT;
    }
}
```

### 2.8 request-reply

**生产者**

```java
@Service
public class ProducerService {

    @Autowired
    RocketMQTemplate rocketMQTemplate;

    public void sendMessage() {
        // 第三个参数为返回值类型
        ReturnType response = rocketMQTemplate.sendAndReceive("SimpleTopic",
                                                              MessageBuilder.withPayload("hello request").build(),
                                                              ReturnType.class
                                                             );
        System.out.println(response);
    }
}
```

**消费者**

```java
@Service
@RocketMQMessageListener(topic = "SimpleTopic", consumerGroup = "${rocketmq.consumer.group}")
// 实现 RocketMQReplyListener 接口
public class ConsumerService implements RocketMQReplyListener<String, ReturnType> {

    @Autowired
    RocketMQTemplate rocketMQTemplate;

    @Override
    public ReturnType onMessage(String message) {
        System.out.println("received message:" + message);
        return new ReturnType("success");
    }
}
```

### 2.9 消费重试时间间隔&自定义是否进入死信队列配置

rocketmq 默认消息发送 16 次后数据会自动进入死信队列，每次重试的偏移量会在重试队列记录，修改重试的时间间隔以及自定义是否直接进入死信队列这些配置再 springboot 中暂时没有提供可用的接口，官方建议使用默认的方式，如果有此业务需求,可以通过支持原生Listener的使用方式自己控制`ConsumeConcurrentlyStatus`实现，具体实现代码如下：

```java
/**
 * RocketMQ Push模式消费
 */
@Service
@RocketMQMessageListener(
    topic = "${demo.rocketmq.topic}",
    consumerGroup = "${demo.rocketmq.consumerGroup}",
    selectorExpression = "${demo.rocketmq.selector-expression}",
    selectorType = SelectorType.TAG,
    messageModel =  MessageModel.CLUSTERING
    // accessKey = "znb00004", // It will read by `rocketmq.consumer.access-key` key
    // secretKey = "12345678" // It will read by `rocketmq.consumer.secret-key` key
)
public class ACLStringPushConsumer implements RocketMQPushConsumerLifecycleListener {
    @Autowired
    private MyMessageListenerConcurrently myMyMessageListenerConcurrently;
    /**
     * 对消费者客户端的一些配置
     * @param consumer
     */
    @Override
    public void prepareStart(DefaultMQPushConsumer consumer) {
        //设置从当前时间开始消费
        consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_TIMESTAMP);
        System.out.println(UtilAll.timeMillisToHumanString3(System.currentTimeMillis()));
        consumer.setConsumeTimestamp(UtilAll.timeMillisToHumanString3(System.currentTimeMillis()));
        //设置最大重试次数.默认16次
        consumer.setMaxReconsumeTimes(10);
        //配置重试消息逻辑,默认是context.setDelayLevelWhenNextConsume(0);
        consumer.setMessageListener(myMyMessageListenerConcurrently);
    }
}
```

自定义消息监听器 MyMessageListenerConcurrently 实现：

```java
@Service
public class MyMessageListenerConcurrently implements MessageListenerConcurrently {

    private static final Logger log = LoggerFactory.getLogger(MyMessageListenerConcurrently.class);

    @Override
    public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs, ConsumeConcurrentlyContext context) {
        //1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h
        // 0表示每次按照上面定义的时间依次递增,第一次为10s,第二次为30s...
        //-1表示直接发往死信队列,不经过重试队列.
        //>0表示每次重试的时间间隔,由我们用户自定义,1表示重试间隔为1s,2表示5s,3表示10秒,依次递增,重试次数由配置consumer.setMaxReconsumeTimes(10)决定
        //发送的默认重试队列topic名称为%RETRY%+消费者组名,发送的默认死信队列topic名称为%DLQ%+消费者组名
        context.setDelayLevelWhenNextConsume(1); //表示重试间隔为1s
        MessageExt msg = msgs.get(0);
        log.debug("received msg: {}", msg);
        try {
            System.out.printf("%s Receive New Messages: %s %n", Thread.currentThread().getName(), msgs);
            int i = 1 / 0;
            System.out.println(i);

        } catch (Exception e) {
            log.warn("consume message failed. messageExt:{}", msg, e);
            long d = System.currentTimeMillis();
            SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
            System.out.println("当前时间:" + sdf.format(d));
            if (msgs.get(0).getReconsumeTimes() > 3) {
                context.setDelayLevelWhenNextConsume(-1); //重试大于3次直接发往死信队列
            }
            return ConsumeConcurrentlyStatus.RECONSUME_LATER;
        }
        return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
    }
}

```

### 2.10 过滤消息

#### 1.根据Tag

消息发送端只能设置一个 Tag，接收端可以设置接收多个。

**生产者**

```java
public void sendMessage() {
    Message<String> message1 = MessageBuilder.withPayload("testTAG1").build();
    Message<String> message2 = MessageBuilder.withPayload("testTAG2").build();
    Message<String> message3 = MessageBuilder.withPayload("testTAG3").build();
    // 使用':'连接主题和 TAG
    rocketMQTemplate.convertAndSend("SimpleTopic:" + "TAG1", message1);
    rocketMQTemplate.convertAndSend("SimpleTopic:" + "TAG2", message2);
    rocketMQTemplate.convertAndSend("SimpleTopic:" + "TAG3", message3);
}
```

**消费者**

```java
@Service
// 使用 selectorExpression 和 selectorType 属性
@RocketMQMessageListener(topic = "SimpleTopic",
                         consumerGroup = "${rocketmq.consumer.group}",
                         selectorExpression = "TAG1 || TAG2",
                         selectorType = SelectorType.TAG)
public class ConsumerService implements RocketMQListener<String> {

    @Override
    public void onMessage(String message) {
        System.out.println("received message:" + message);
    }
}
```

#### 2.根据SQL表达式

需要开启支持 SQL 表达式，在 broker.conf 中添加如下配置：

```bash
enableProertyFilter = true
```

## 3.集群

修改 nameServ

```properties
rocketmq.name-server=192.168.11.101:9876;192.168.11.102:9876
```

