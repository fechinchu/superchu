#  消息中间件01-RabbitMQ

# 1.RabbitMQ

## 1.1.RabbitMQ介绍

MQ全称为Message Queue，即消息队列，RabbitMQ是由erlang语言开发，基AMQP(Advanced Message Queue 高级消息队列协议)协议实现的消息队列，它是一种应用程序之间的通信方法，消息队列在分布式系统开发中应用十分广泛。

**AMQP和JMS的区别和联系:**

* JMS是定义了统一的接口,来对消息操作进行同一,AMQP是用过规定协议来统一数据交互. 

* JMS限定了必须使用Java语言,AMQP只是协议,不规定时限方式,因此是跨语言的.
*  JMS规定了两种消息模型,而AMQP的消息模型更加丰富.

**常见的MQ产品:**

* ActiveMQ:基于JMS,Apache 
* RabbitMQ:基于AMQP协议,erlang语言开发,稳定性好. 
* RocketMQ:基于JMS,阿里巴巴产品,目前交由Apache基金会. 
* Kafka:分布式消息系统,高吞吐量.

> 1:如果解决消息丢失?
>
> * ack(消费者确认):通过消息确认(Acknowlege)机制来实现,当消费者获取消息后,会向RabbitMQ发送执行 ACK,告知消息已经被接收,不过这种ACK分两种情况:
>   * 自动ACK:消息一旦被接收,消费者自动发送ACK;
>   * 手动ACK:消息接收后,不会发送ACK.需要手动调用; 
> * 持久化:我们需要将消息持久化到硬盘,防止服务宕机导致消息丢失.设置消息持久化,前提是:Queue,Exchange都持久化.Exchange,Queue,message持久化;
> * 生产者确认:生产者发送消息后,等待mq的ACK,如果没有收到或者收到失败消息,那么就会重试,如果收到成 功消息那么业务结束; 
> * 可靠消息服务:对于部分不支持生产者确认的消息队列,可以发送消息前,将消息持久化到数据库,并记录消息 状态,后续消息发送,消费等过程都依赖于数据库中消息状态的判断和修改。
>
> 2:如何避免消息堆积?
>
> * 通过同一队列多消费者监听,实现消息的争抢,加快消息消费速度。
>
> 3:如何保证消息的有序性?
>
> * 保证消息发送时候有序同步发送;
> * 保证消息发送被同一队列接收;
> * 保证一个队列只有一个消费者,可以有从机(待机状态),实现高可用。
>
> 4:如何保证避免消息重复消费?
>
> * 如果,你拿到这个消息做数据库的insert操作,那就容易了,给这个消息做一个唯一主键,那么就算出现了重复 消费的情况,就会导致主键冲突;
> * 如果拿到这个消息做Redis的set操作,不用解决,无论set几次结果都是一样,set操作本来就是幂等操作;
> * 如果上述的情况不行,可以准备一个三方介质,来做消费记录,以Redis为例,给消费分配一个全局id,只要消费 过该消息,将<id,Message>以K-V形式写入redis,那消费者开始消费前,先去redis中查询有没有消费记录即可。

**RabbitMQ的原理**

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210707162748530.png" alt="image-20210707162748530" style="zoom:50%;" />

**组成部分说明：**

> * Broker:消息队列服务进程，此进程包括两个部分:Exchange和Queue；
> * Exchange:消息队列交换机，按一定的规则将消息路由转发到某个队列，对消息进行过滤；
> * Queue:消息队列，存储消息的队列，消息到达队列并转发给指定的消费方；
> * Producer:消息生产者，即生产方客户端，生产方客户端将消息发送到MQ；
> * Consumer:消息消费者，即消费方客户端，接收MQ转发的消息。

> Exchange:交换机,一方面,接收生产者发送的消息,知道如何处理消息,例如递交给某个特别的队列,递交给所有的队列 等.到底如何操作,取决于Exchange的型.Exchange有以下3种类型.
>
> * Fanout:广播,将消息交给所有绑定到交换机的队列.
> * Direct:定向,把消息交给符合指定的routing key的队列. 
> * Topic:通配符,把消息交给符合Routing pattern(路由模式)的队列.
>
> **Exchange只负责转发消息,不具备存储消息的能力**,因此如果没有任何队列Exchange绑定,或者没有符合路由规则的队列,那么消息会丢失。

**消息发布接收流程：**

> **发送消息**
>
> 1. 生产者和Broker建立TCP连接；
> 2. 生产者和Broker建立通道；
> 3. 生产者通过通道消息发送给Broker，由Exchange将消息进行转发；
> 4. Exchange将消息转发到指定的Queue(队列)。
>
> **接收消息**
>
> 1. 消费者和Broker建立TCP连接；
> 2. 消费者和Broker建立通道；
> 3. 消费者监听指定的Queue(队列)；
> 4. 当有消息到达Queue时Broker默认将消息推送给消费者；
> 5. 消费者接收到消息。

## 1.2.RabbitMQ的工作模式

RabbitMQ提供了6种消息模型**

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210707162801325.png" alt="image-20210707162801325" style="zoom:50%;" />

![image-20191225151309583](../微服务笔记/Education项目总结 05.assets/image-20191225151309583.png)

### 1.2.1.Work queues

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210707162808723.png" alt="image-20210707162808723" style="zoom: 67%;" />

Work queues与入门程序相比，多了一个消费端，两个消费端共同消费同一个队列中的消息。

**应用场景**：对于任务过重或任务较多的情况使用工作队列可以提高任务的速度。

**测试:** 

1. 使用入门程序，启动多个消费者；

2. 生产者发送多个消息。

**结果：**

1. 一条消息只会被一个消费者接收
2. rabbitmq采用轮询的方式将消息平均发送给消费者；
3. 消费者在处理完某条消息后，才会收到下一条消息。

### 1.2.2.Publish/Subscribe

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210707162817694.png" alt="image-20210707162817694" style="zoom: 67%;" />

**发布订阅模式**：

1. 每个消费者监听自己的队列；
2. 生产者将消息发给broker，由交换机将消息转发到绑定此交换机的每个队列，每个绑定交换机的队列都将接收到消息；

**应用场景：**当用户充值成功或转账完成系统通知用户，通知方式有短信和邮件多种方法。

**结果：**使用生产者发送若干条消息，每条消息都转发到各个队列，每个消费者都接收到了消息。

### 1.2.3.Routing

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210707162836277.png" alt="image-20210707162836277" style="zoom: 50%;" />

**路由模式：**

1. 每个消费者监听自己的队列，并且设置routingkey；
2. 生产者将消息发送给交换机，由交换机根据routingkey来转发到指定的队列；

### 1.2.4.Topics

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210707162851456.png" alt="image-20210707162851456" style="zoom: 50%;" />

**通配符工作模式：**

1. Topics和Routing的基本原理相同，即生产者将详细发送给交换机，交换机根据routingKey将消息转发给与routingKey匹配的队列；
2. Topics和Routing的基本原理相同,Routing模式是相等匹配，topics模式是通配符匹配；
   1. 符号#:匹配零个或者多个词（每个词中间以`.`进行分割），比如`inform.#`可以匹配inform.sms, inform.email，inform.message.sms；
   2. 符号*:只能匹配一个词,比如`info.\*`可以匹配inform.sms,inform.email;

**应用场景：**

1. 根据用户的通知设置去通知用户，设置接收Email的用户只接收Email,设置接收sms的用户只接收sms，设置两种通知类型都接收的则两种通知都有效。

### 1.2.5.Header

**工作模式：**

1. Header模式与routing不同的地方在于，header模式取消routingkey,使用header中的key/value(键值对)匹配队列。

**生产者代码：**

~~~java
Map<String, Object> headers_email = new Hashtable<String, Object>();
headers_email.put("inform_type", "email");
Map<String, Object> headers_sms = new Hashtable<String, Object>();
headers_sms.put("inform_type", "sms");
channel.queueBind(QUEUE_INFORM_EMAIL,EXCHANGE_HEADERS_INFORM,"",headers_email); channel.queueBind(QUEUE_INFORM_SMS,EXCHANGE_HEADERS_INFORM,"",headers_sms);
~~~

**消费者代码**

~~~java
String message = "email inform to user"+i;
Map<String,Object> headers = new Hashtable<String, Object>();
headers.put("inform_type", "email");//匹配email通知消费者绑定的header
//headers.put("inform_type", "sms");//匹配sms通知消费者绑定的header AMQP.BasicProperties.Builder properties = new AMQP.BasicProperties.Builder();
properties.headers(headers);
//Email通知
channel.basicPublish(EXCHANGE_HEADERS_INFORM, "", properties.build(), message.getBytes());
~~~

### 1.2.6.RPC

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20210707162908259.png" alt="image-20210707162908259" style="zoom: 50%;" />

RPC即客户端远程调用服务端的方法，使用MQ可以实现RPC的异步调用，基于Direct交换机实现，流程如下：

1. 客户端即是生产者就是消费者，向RPC请求队列发送RPC调用消息，同时监听RPC响应队列；
2. 服务端监听RPC请求队列的消息，收到消息后执行服务端的方法，得到方法返回的结果；
3. 服务端将RPC方法的结果发送给RPC响应队列；
4. 客户端（RPC调用方）监听RPC响应队列，接收到RPC调用结果。

## 1.3.代码实现

### 1.3.1.配置类

```java
@Configuration
public class RabbitMQConfig {
    /**
     * 队列名称
     */
    public static final String QUEUE_INFORM_EMAIL = "queue_inform_email";
    public static final String QUEUE_INFORM_SMS = "queue_inform_sms";
    /**
     * 交换机名称
     */
    public static final String EXCHANGE_TOPICS_INFORM = "exchange_topics_inform";
    /**
     * 路由名称
     */
    public static final String ROUTING_KEY_EMAIL = "inform.#.email.#";
    public static final String ROUTING_KEY_SMS = "inform.#.sms.#";
    //声明交换机
    @Bean(EXCHANGE_TOPICS_INFORM)
    public Exchange exchange(){
        //durable(持久化)，mq重启之后交换机还在；
        return ExchangeBuilder.topicExchange(EXCHANGE_TOPICS_INFORM).durable(true).build();
    }
    //声明队列
    @Bean(QUEUE_INFORM_EMAIL)
    public Queue queueEmail(){
        return new Queue(QUEUE_INFORM_EMAIL);
    }

    @Bean(QUEUE_INFORM_SMS)
    public Queue queueSMS(){
        return new Queue(QUEUE_INFORM_SMS);
    }

    //绑定队列和交换机
    @Bean
    public Binding bindingEmail(@Qualifier(QUEUE_INFORM_EMAIL) Queue queue,
                           @Qualifier(EXCHANGE_TOPICS_INFORM)Exchange exchange){
        return BindingBuilder.bind(queue).to(exchange).with(ROUTING_KEY_EMAIL).noargs();
    }
    @Bean
    public Binding bindingSMS(@Qualifier(QUEUE_INFORM_SMS) Queue queue,
                           @Qualifier(EXCHANGE_TOPICS_INFORM)Exchange exchange){
        return BindingBuilder.bind(queue).to(exchange).with(ROUTING_KEY_SMS).noargs();
    }
}
```

### 1.3.2.消息生产者

```java
@SpringBootTest
@RunWith(SpringRunner.class)
public class Producer05TopicsSpringBoot {

    @Autowired
    private RabbitTemplate rabbitTemplate;

    public String message = "send message to fechin ";
    @Test
    public void testSendEmail(){
        /**
         * 参数
         * 1.交换机名称
         * 2.路由key
         * 3.消息内容
         */
        rabbitTemplate.convertAndSend(RabbitMQConfig.EXCHANGE_TOPICS_INFORM,
                "inform.email",message);
    }
}
```

### 1.3.3.消息消费者

```java
@Component
public class ReceiveHandler {

    /**
     * 监听email队列
     * @param msg
     * @param message
     * @param channel
     */
    @RabbitListener(queues = {RabbitMQConfig.QUEUE_INFORM_EMAIL})
    public void receive_email(String msg, Message message, Channel channel){
        System.out.println(msg);
    }

    /**
     * 监听sms队列
     * @param msg
     * @param message
     * @param channel
     */
    @RabbitListener(queues = {RabbitMQConfig.QUEUE_INFORM_SMS})
    public void receive_sms(String msg,Message message,Channel channel){
        System.out.println(msg);
    }
}
```

# 
