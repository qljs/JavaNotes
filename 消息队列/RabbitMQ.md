

## 一  消息可靠性



### 1. 生产者消息可靠性

生产者提供了两种方式，一种是事务机制，  但是性能比较差；另一种是 confirm 模式。

#### 1.1 开启 confirm 机制

RabbitMQ 整个消息的投递顺序为：生产者 ——> 交换机——> 队列 ——> 消费者。

- **消息从消费者到交换机，通过回调 comfirmCallback 可以保证消息投递成功**。
- **消息从交换机失败，会回调 returnCallback 来进行处理。**

SpringBoot中的配置

```properties
# 开启confrim
spring.rabbitmq.publisher-confirms=true
# 开启return
spring.rabbitmq.publisher-returns=true
```

Spring 下的配置

```java
// 开启return
connectionFactoryfactory.setPublisherReturns(true);
// 开启confrim
// CORRELATED：通过获取CorrelationData
// SIMPLE：通过waitForConfirms()或者waitForConfirmsOrDie获取
connectionFactoryfactory.setPublisherConfirmType(CachingConnectionFactory.ConfirmType.CORRELATED);
```















## 参考

https://docs.spring.io/spring-amqp/docs/2.1.10.RELEASE/reference/html/#template-confirms