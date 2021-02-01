

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

**RabbitMQ对于queue中的message的保存方式有两种方式：disc（磁盘）和ram（内存）。**

### 2.1 消息持久化触发

- 消息发送时设置了持久化`deliverMode=2`；
- 当 RabbitMQ 内存紧张导致 RAM 即将用尽时，会将消息持久化到磁盘，释放 RAM。

**触发消息刷新到磁盘**

1. 写入文件前会有一个Buffer，大小为1M，数据在写入文件时，首先会写入到这个Buffer，如果Buffer已满，则会将Buffer写入到文件（未必刷到磁盘）；
2. 有个固定的刷盘时间：25ms，也就是不管Buffer满不满，每隔25ms，Buffer里的数据及未刷新到磁盘的文件内容必定会刷到磁盘；
3. 每次消息写入后，如果没有后续写入请求，则会直接将已写入的消息刷到磁盘：使用Erlang的receive x after 0来实现，只要进程的信箱里没有消息，则产生一个timeout消息，而timeout会触发刷盘操作。





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





## 四 消息堆积

1.排查消费者的消费性能瓶颈

2.增加消费者的多线程处理

3.部署增加多个消费者

4.新增消息队列，可以想办法把消息按顺序的转移到一个新的队列，让消费者消费新队列中的消息

如果出现大规模消息积压无法解决，可以采用以下思路：

- 新建几倍的消息队列，写一个临时的程序，把消息分发到这些消息队列里；
- 新建几倍的consumer，用来消费上述新建的队列；
- 消费完成之后，重新恢复原来架构。





## 五 Push和Pull

- Push 模式：当接收到生产者发送的消息后，立即推送给消费者；
- Pull 模式：服务端收到消息后，等着消费者主动来拉取。

Push 模式和 Pull 模式区别：

- Push 模式的实时性较好，而 Pull 采用长轮询，会有轮询间隔，实时性较差；
- Push 模式会存在慢消费的问题，在消费者消费速度较慢时，broker还是会不断推送消息给消费者，导致消费者无法处理，进而重发；而pull模式，consumer可以按需消费，不用担心自己处理不了的消息来骚扰自己，而broker堆积消息也会相对简单，无需记录每一个要发送消息的状态，只需要维护所有消息的队列和偏移量就可以了。





## 参考

https://docs.spring.io/spring-amqp/docs/2.1.10.RELEASE/reference/html/#template-confirms

https://tech.meituan.com/2016/07/01/mq-design.html