

## 一  消息可靠性



### 1. 生产者消息可靠性

生产者提供了两种方式，一种是事务机制，  但是性能比较差；另一种是 confirm 模式。

#### 1.1 开启 confirm 机制

RabbitMQ 整个消息的投递顺序为：生产者 ——> 交换机——> 队列 ——> 消费者。

- **消息从消费者到交换机，通过回调 comfirmCallback 可以保证消息投递成功**。
- **消息从交换机失败，会回调 returnCallback 来进行处理。**

**`confirm`机制是异步的，成功时回调会返回`true`，失败时会返回`false`。事务机制是同步的**

SpringBoot中的配置

```properties
# 开启confrim
spring.rabbitmq.publisher-confirms=true
# 开启return
spring.rabbitmq.publisher-returns=true
```

Spring 下的配置

```java
// 开启return，消息路由不到交换机或者队列时，调用该回调
connectionFactoryfactory.setPublisherReturns(true);
// 开启confrim
// CORRELATED：通过获取CorrelationData
// SIMPLE：通过waitForConfirms()或者waitForConfirmsOrDie获取
connectionFactoryfactory.setPublisherConfirmType(CachingConnectionFactory.ConfirmType.CORRELATED);
```



#### 1.2 消息落地入库，进行重试

虽然 RabbitMQ 对生产者端提供了 confirm 机制，但是 confirm 机制是异步的，而且因为网络波动等原因，消息已经投递成功，但是回调失败，为了保证可靠性，可以将消息入库，进行重试，但这会带来重复消费，幂等问题。

**消息落地入库第一种方案：**

在业务数据入库时，消息同时落地入库，但是因为要操作两个数据库，而且若是分布式环境下，还要考虑分布式事务问题，性能较差。



![](D:\JavaNotes\JavaNotes\images\mq\msg1.png)



**消息落地入库第二种方案：**

第二种方案，根据消费者发送的确认消息，来进行消息入库，然后根生产者发送演示检查消息，来判断消息是否成功或者客户端是否发送确认消息。

![](D:\JavaNotes\JavaNotes\images\mq\msg2.png)



> #### 死信队列

死信是 RabbitMQ 中的一种消息机制，当消息符合以下几种情况时，会变成死信。

- 消息被拒绝。`basicNack`或`basicReject`且`requeue`属性设置为 fasle；
- 消息设置了存活时间，或者队列设置了消息过期时间`x-message-ttl`；
- 队列中信息已满，即达到队列最大长度。

```java
public Queue topicQueue(){
    Map<String, Object> args = new HashMap<>();
    // 设置消息发送的死信队列
    args.put("x-dead-letter-exchange","deadLetterExchange");
    // 发送队列时路由key
    args.put("x-dead-letter-routing-key","deadLetter");
    // 队列消息过期时间，单位毫秒
    args.put("x-message-ttl",10000);
    Queue queue = new Queue("topicQueue", true, false, false, args);
    return queue;
}
```



### 2. 队列和消息持久化

在 RabbitMQ 中，通过`durable=true`可以设置对立的持久化，通过`deliveryMode = PERSISTENT`可以设置消息的持久化。



### 3. 消费者ACK机制

在 RabbitMQ 中，默认消费者自动消费消息，取消自动消费后，需要手动调用`ack`或者`nack`通知 RabbitMQ。

```java
// deliveryTag：消息的标识
// multiple：是否批量消费
channel.basicAck(message.getMessageProperties().getDeliveryTag(),false);
```

通过配置参数可以设置每次消费的数量：

```properties
spring.rabbitmq.listener.prefetch=1
```



## 二 消息幂等性

为了保证消息的可靠性，采用了重发的策略，或者因为消费者在`ack`之前出现问题，导致没有确认回复，都会导致一条消息被消费多次。

确保消息幂等性的一些办法：

- 利用 Redis 的分布式锁；
- 生产者发送消息时，添加一个全局唯一ID，在消费前对比下是否消费过。



## 三 消息的顺序性

### 1. 一个队列对应一个消费者

当一个队列对应多个消费者时，不同消费者消费的速度不同，可能导致最终结果乱掉。此时可以让一个消费者对应一个队列，来保证消息的顺序性。

这种办法也存在问题，对于生产者端，若是开启 confirm 机制，通常为了保证消息可靠性，会进行重试，那在生产者在消息重试的时候，消息的顺序可能已经乱掉了。对于消费者，若使用了 ack，在消费失败后，同样也无法保证消息的顺序性。

这种方式对于集群部署不适用，很难确保一个消费者对应一个队列。



### 2. 对消息进行编号

在发送消息时为消息加上序号，消费者根据顺序进行消费，。













## 参考

https://docs.spring.io/spring-amqp/docs/2.1.10.RELEASE/reference/html/#template-confirms

https://tech.meituan.com/2016/07/01/mq-design.html