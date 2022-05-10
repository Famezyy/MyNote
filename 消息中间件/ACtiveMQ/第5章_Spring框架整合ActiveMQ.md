# 第5章_Spring框架整合ActiveMQ

## 1.Spring整合ActiveMQ

### 1.1 pom.xml添加依赖

```xml
<dependencies>
   <!-- activemq核心依赖包  -->
    <dependency>
        <groupId>org.apache.activemq</groupId>
        <artifactId>activemq-all</artifactId>
        <version>5.10.0</version>
    </dependency>
    <!--  嵌入式activemq的broker所需要的依赖包   -->
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
        <version>2.10.1</version>
    </dependency>
    <!-- activemq连接池 -->
    <dependency>
        <groupId>org.apache.activemq</groupId>
        <artifactId>activemq-pool</artifactId>
        <version>5.15.10</version>
    </dependency>
    <!-- spring支持jms的包 -->
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-jms</artifactId>
        <version>5.2.1.RELEASE</version>
    </dependency>
    <!--spring相关依赖包-->
    <dependency>
        <groupId>org.apache.xbean</groupId>
        <artifactId>xbean-spring</artifactId>
        <version>4.15</version>
    </dependency>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-aop</artifactId>
        <version>5.2.1.RELEASE</version>
    </dependency>
    <!-- Spring核心依赖 -->
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-core</artifactId>
        <version>4.3.23.RELEASE</version>
    </dependency>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-context</artifactId>
        <version>4.3.23.RELEASE</version>
    </dependency>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-aop</artifactId>
        <version>4.3.23.RELEASE</version>
    </dependency>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-orm</artifactId>
        <version>4.3.23.RELEASE</version>
    </dependency>
</dependencies>
```

### 1.2 Spring的ActiveMQ配置文件

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd http://www.springframework.org/schema/context https://www.springframework.org/schema/context/spring-context.xsd">

    <!-- 开启包的自动扫描 -->
    <context:component-scan base-package="com.activemq.demo"/>
    <!-- 配置生产者 -->
    <bean id="jmsFactory" class="org.apache.activemq.pool.PooledConnectionFactory" destroy-method="stop">
        <property name="connectionFactory">
            <!-- 真正可以生产Connection的ConnectionFactory，由对应的JMS服务商提供 -->
            <bean class="org.apache.activemq.spring.ActiveMQConnectionFactory">
                <property name="brokerURL" value="tcp://192.168.11.101:61616"/>
            </bean>
        </property>
        <property name="maxConnections" value="100"/>
    </bean>

    <!-- 这个是队列目的地，点对点的Queue -->
    <bean id="destinationQueue" class="org.apache.activemq.command.ActiveMQQueue">
        <!-- 通过构造注入Queue名 -->
        <constructor-arg index="0" value="spring-active-queue"/>
    </bean>

    <!-- 这个是主题目的地，发布订阅的主题Topic -->
    <bean id="destinationTopic" class="org.apache.activemq.command.ActiveMQTopic">
        <constructor-arg index="0" value="spring-active-topic"/>
    </bean>

    <!-- Spring提供的JMS工具类，他可以进行消息发送,接收等 -->
    <bean id="jmsTemplate" class="org.springframework.jms.core.JmsTemplate">
        <!-- 传入连接工厂 -->
        <property name="connectionFactory" ref="jmsFactory"/>
        <!-- 传入目的地 -->
        <property name="defaultDestination" ref="destinationQueue"/>
        <!-- 消息自动转换器 -->
        <property name="messageConverter">
            <bean class="org.springframework.jms.support.converter.SimpleMessageConverter"/>
        </property>
    </bean>
</beans>
```

### 1.3 队列生产者

```java
@Service
public class SpringProduce {

    @Autowired
    private JmsTemplate jmsTemplate;

    public static void main(String[] args) {
        ApplicationContext applicationContext = new ClassPathXmlApplicationContext("applicationContext.xml");
        SpringProduce springProduce = applicationContext.getBean(SpringProduce.class);
        springProduce.jmsTemplate.send(
                new MessageCreator() {
                    public Message createMessage(Session session) throws JMSException {
                        return session.createTextMessage("***Spring和ActiveMQ的整合case111.....");
                    }
                }
        );
        System.out.println("********send task over");
    }
}
```

### 1.4 队列消费者

```java
@Service
public class SpringConsumer {

    @Autowired
    private JmsTemplate jmsTemplate;

    public static void main(String[] args) {
        ApplicationContext applicationContext = new ClassPathXmlApplicationContext("applicationContext.xml");
        SpringConsumer springConsumer = applicationContext.getBean(SpringConsumer.class);
        String message = (String)springConsumer.jmsTemplate.receiveAndConvert();
        System.out.println("********收到的消息为：" + message);
    }
}
```

### 1.5 主题生产者和消费者

只需修改 xml 配置文件

```xml
<bean id="jmsTemplate" class="org.springframework.jms.core.JmsTemplate">
    <property name="connectionFactory" ref="jmsFactory"/>
    <!-- 传入主题的目的地 -->
    <property name="defaultDestination" ref="destinationTopic"/>
    <property name="messageConverter">
        <bean class="org.springframework.jms.support.converter.SimpleMessageConverter"/>
    </property>
</bean>
```

### 1.6 配置消费者的监听类

修改 xml 文件，添加一个监听器组件

```xml
<bean id="jmbContainer" class="org.springframework.jms.listener.DefaultMessageListenerContainer">
	<property name="connectionFactory" ref="jmsFactory"></property>
    <!-- 需要与 jsmTemplate 的目的地一致 -->
    <property name="destination" ref="destinationTopic"></property>
    <property name="messageListener" ref="myMessageListener"></property>
</bean>
```

创建一个 messageListener

```java
@Component
public class MyMessageListener implements MessageListener {
    @Override
    public void onMessage(Message message) {
        if (message instanceof TextMessage) {
            TextMessage textMessage = (TextMessage) message;
            try {
                System.out.println(textMessage.getText());
            } catch (JMSException e) {
                e.printStackTrace();
            }
        }
    }
}
```

启动生产者，监听器会自动监听消息并打印出来

## 2.SpringBoot整合ActiveMQ

### 2.1 队列生产者

添加依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-activemq</artifactId>
    <version>2.6.7</version>
</dependency>
```

配置 application.yml

```yaml
spring:
  activemq:
    broker-url: tcp://192.168.11.101:61616
    user: admin
    password: admin
  jms:
    pub-sub-domain: false # false = Queue true = topic

# 配置 queue 名称
myQueue: boot-activemq-queue
```

配置 configBean

```java
@Configuration
// 允许 Jms
@EnableJms
public class ActivemqConfig {

    @Value("${myQueue}")
    private String myQueue;

    @Bean
    public Queue queue() {
        return new ActiveMQQueue(myQueue);
    }
}
```

生产者类

```java
@Component
public class QueueProduce {

    @Autowired
    private JmsMessagingTemplate jmsMessagingTemplate;

    @Autowired
    private Queue queue;

    public void produceMsg() {
        jmsMessagingTemplate.convertAndSend(queue, "***** This is a message *****");
    }
}
```

测试

```java
@SpringBootTest
public class TestActivemq {

    @Autowired
    private QueueProduce queueProduce;

    @Test
    void testSend() {
        queueProduce.produceMsg();
    }
}
```

### 2.2 设置定时任务

在生产者的发送消息方法上添加`@Scheduled`注解

```java
// 间隔3秒定时投放
@Scheduled(fixedDelay = 3000)
public void produceMsgScheduled() {
    jmsMessagingTemplate.convertAndSend(queue, "***** This is a scheduled message *****");
    System.out.println("send a scheduled message");
}
```

在配置类上添加`@EnableScheduling`注解开启计时功能

```java
@Configuration
@EnableJms
@EnableScheduling
public class ActivemqConfig {
```

启动主程序，观察消息发送的过程

### 2.3 队列消费者

```java
@Component
public class QueueConsumer {
    @JmsListener(destination = "${myQueue}")
    public void receive(TextMessage textMessage) throws JMSException {
        System.out.println("received a message：" + textMessage.getText());
    }
}
```

### 2.4 主题生产者

配置 application.yaml 文件

```yaml
spring:
  activemq:
    broker-url: tcp://192.168.11.101:61616
    user: admin
    password: admin
  jms:
    pub-sub-domain: true # false = Queue true = topic
# 配置 topic 名称
myTopic: boot-activemq-topic
```

修改配置类

```java
@Configuration
@EnableJms
@EnableScheduling
public class ActivemqConfig {

    @Value("${myTopic}")
    private String myTopic;

    @Bean
    public Topic topic() {
        return new ActiveMQTopic(myTopic);
    }
}
```

生产者类

```java
@Component
public class TopicProduce {
    @Autowired
    private JmsMessagingTemplate jmsMessagingTemplate;

    @Autowired
    private Topic topic;

    @Scheduled(fixedDelay = 3000)
    public void produceTopic() {
        jmsMessagingTemplate.convertAndSend(topic, "*** This is a topic ***");
    }
}
```

### 2.5 主题消费者

同队列消费者，作用都是监听消息

```java
@Component
public class TopicConsumer {
    @JmsListener(destination = "${myTopic}")
    public void receive(TextMessage textMessage) throws JMSException {
        System.out.println("received a topic：" + textMessage.getText());
    }
}
```

**持久化订阅者**

修改配置类

```java
@Configuration
@EnableJms
@EnableScheduling
public class ActivemqConfig {

    @Value("${myTopic}")
    private String myTopic;

    @Bean
    public Topic topic() {
        return new ActiveMQTopic(myTopic);
    }

    // 修改 DefaultJmsListenerContainerFactory
    @Bean
    public DefaultJmsListenerContainerFactory jmsListenerContainerFactory(SingleConnectionFactory singleConnectionFactory) {
        DefaultJmsListenerContainerFactory containerFactory = new DefaultJmsListenerContainerFactory();
        singleConnectionFactory.setClientId("TEST1");
        containerFactory.setConnectionFactory(singleConnectionFactory);
        // 设置为永久订阅者
        containerFactory.setSubscriptionDurable(true);
        return containerFactory;
    }
}
```

修改消费者类

```java
@Component
public class TopicConsumer {
    // 指定 containerFactory
    @JmsListener(subscription = "sub1", destination = "${myTopic}", containerFactory = "jmsListenerContainerFactory")
    public void receive(TextMessage textMessage) throws JMSException {
        System.out.println("received a topic：" + textMessage.getText());
    }
}
```

