## 一 基本概念

### 1. 简介

Kafka 是一个分布式事件流 (even stream) 服务，Kafka强依赖于 zk。

流平台具有三个关键功能：

1. **消息队列**：发布和订阅消息流，这个功能类似于消息队列，这也是 Kafka 也被归类为消息队列的原因。
2. **容错的持久方式存储记录消息流**： Kafka 会把消息持久化到磁盘，有效避免了消息丢失的风险·。
3. **流式处理平台：** 在消息发布的时候进行处理，Kafka 提供了一个完整的流式处理类库。

Kafka 主要有两大应用场景：

1. **消息队列** ：建立实时流数据管道，以可靠地在系统或应用程序之间获取数据。
2. **数据处理：** 构建实时的流数据处理程序来转换或处理数据流。

![image-20210125114300706](D:\JavaNotes\JavaNotes\images\mq\kafka-1.png)



### 2. 基本概念

> #### Topic(主题) 和 Logs(消息日志)

Topic 是一个抽象概念，是接收消息的类别名称，对同一个 Topic，下面可以有多个分区（Partition）日志文件，每个分区都是有一个有序的消息序列，这些消息会按顺序写入到 commit log 文件，每个分区的每条消息都有唯一编号（offset），用来标识分区中的消息记录。

**在 kafka 中被消费的消息不会被删除，而是根据配置，决定什么时候删除。每个消费者自己维护 offset，并基于自己的 commitlog 进行消费。**

![](..\images\mq\kafka-topic.png)

> #### Partition（分区）

每个 Topic 中可以有多个分区，类似于其他消息中间件中的队列。



> #### Producer（生产者）、Consumer（消费者）、ConsumerGroup（消费者组）

与其他消息中间件一样。在 kafka 中，消费者有两种不同的模式：

- queue 模式，所有的消费者都在同一个 consumer group 下，消息只会被其中一个消费者消费；
- publish-subscrebe 模式：所有消费者有自己的 consumer group，消息可以被不同消费组的一个消费者消费。



> #### Broker

消息中间处理节点。



> #### Distribution（分布式）

log 的分区分布集群中不同的 broker 上，每个 broker 都可以备份其他 broker 中分区的数据。对于每个分区，都有一个 leader 服务节点、零或多个 follower 服务器。**leader 提供读和写，follower 被动的复制 leader，不对外提供服务，在 Leader 挂掉后，follower变成新的 leader。**



## 二 安装配置

### 1. 单机安装

1. 下载后解压安装https://archive.apache.org/dist/kafka/2.2.0/kafka_2.12-2.2.0.tgz

2. 修改配置文件

   ```properties
   # 监听地址和端口
   listeners=PLAINTEXT://172.16.33.52:9092
   # 日志文件地址
   log.dirs=/usr/local/kafka_2.12-2.2.0/log
   # 配置zk
   zookeeper.connect=172.16.33.52:2181
   ```
   
3. 启动zk，然后启动服务

   ```sh
   kafka.server.start.sh [-daemon] server.properties
   ```
   
4. 创建主题 topic

   ```sh
   kafka‐topics.sh --create --zookeeper IP:PORT --replication-factor 1 --partitions 1 --topic topicname
   
   # 查看主题
   bin/kafka‐topics.sh --list --zookeeper IP:PORT
   
   # 删除主题
   bin/kafka‐topics.sh --delete --topic topicname --zookeeper IP:PORT
   ```

5.  测试发送消息

   ```sh
   kafka-console-producer.sh --broker-list IP:PORT --topic topicname
   ```

6. 测试消费消息

   ```sh
   kafka-console-consumer.sh --bootstrap-server IP:PORT --consumer-property group.id=groupID --topic topicname
   ```



## 三 kafka设计

![](..\images\mq\kafka-3.png)



### 1. 核心总控制器 controller

在 kafka 集群中，会有一个 broker 被选举为控制器 (controller) ，它负责管理整个集群的分区和副本：

- 某个分区的 leader 副本出现故障时，重新选举 leader；

- 检测到某个分区的 ISR (已同步副本) 集合发生变化时，通知所有的 broker 更新元数据信息；

- kafka 增加分区时，控制器负责让新分区被其他节点感知到。

  

> #### controller 选举机制

在 kafka 启动时，会选举一台 broker 作为 controller，选举过程是尝试在 zk 上创建临时节点`/controller` ，zk 保证只有一个 broker 创建成功，成功的 broker 成为 controller，其他 broker 监听该临时节点。

当 controller 角色的 broker 宕机时，zk 上的临时节点被删除，触发 zk 的 watch 机制，其他 broker 发现临时节点消失，重新进行竞争选举。

具有控制器的 broker 比其他普通 broker 多了以下的功能：

- **监听 broker 相关的变化**；给 zk  中`/brokers/ids`节点添加监听，用来处理 broker 的增减；
- **监听 topic 相关的变化**；给 zk 中的`/brokers/topics`添加 TopicChangeListener，处理 topic 的增加，给`/admin/delete_topics`节点添加监听，处理 topic 的删除；
- 从 zk 中读取获取当前所有与 topic、partition 以及 broker 有关的信息并进行相应的管理。对于所有 topic 所对应的 zk 中的`/brokers/topics/[topic]`节点添加 PartitionModificationsListener，用来监听 topic中 的分区分配变化。
- 更新集群的元数据信息，同步到其他普通的broker节点中。



> #### 分区副本选举机制

当 controller 感知到分区 leader 所在的 broker 挂掉之后，会从 ISR 列表（配置`unclean.leader.election.enable=false`）中挑选第一个 broker 作为 leader（第一个 broker 最早放入 ISR，可能时同步数据最多的副本），若配置参数为 true，标识 ISR 列表中 broker 都挂了可以在非 ISR 列表的 broker 中选举。

**副本进入 ISR 的条件：**

- 副本节点不能产生分区，必须能与zookeeper保持会话以及跟leader副本网络连通；
- 副本能复制leader上的所有写操作，并且不能落后太多。(与leader副本同步滞后的副本，是由 replica.lag.time.max.ms 配置决定的，超过这个时间都没有跟leader同步过的一次的副本会被移出ISR列表)。



> #### 消费者消费消息的 offset 机制

每个consumer会定期将自己消费分区的offset提交给kafka内部topic：**__consumer_offsets**，提交过去的时候，**key 是 consumerGroupId+topic+分区号，value 就是当前 offset 的值**，kafka 会定期清理 topic 里的消息，最后就保留最新的那条数据

 因为__consumer_offsets 可能会接收高并发的请求，kafka 默认给其**分配50个分区**(可以通过offsets.topic.num.partitions设置)，这样可以通过加机器的方式抗大并发。

通过如下公式可以选出 consumer 消费的 offset 要提交到 __consumer_offsets 的哪个分区

公式：**hash(consumerGroupId)  %  __consumer_offsets主题的分区数**



> #### 消费者 Rebalance 机制

当消费组中的消费者数量或者消费的分区有变化，会触发 kafka 的 rebalance 机制，重新分配消费者和消费分区的关系。**rebalance 机制只针对不指定分区消费的情况，若指定了分区，则不会触发该机制。**

触发消费者 rebalance 的情况：

- 消费组中的消费者数量变化；
- 动态增加了 topic 中的分区；
- 消费组订阅了更多了 topic。

rebalance过程中，消费者无法从kafka消费消息，这对kafka的TPS会有影响，如果kafka集群内节点较多，比如数百个，那重平衡可能会耗时极多，所以应尽量避免在系统高峰期的重平衡发生。



> #### 消费者 Rebalance 分区分配策略

kafka 提供了三种分区分配策略：**range、round-robin、sticky**。通过参数`partition.assignment.strategy`可以设置分区分配策略。

- **range 策略**：按照分区序号排序；例如有m个分区，n个消费者，那么每个消费者分配m/n个分区，剩下的m%n个分区，由前面m%n个消费者分摊；
- **round-robin 策略**：该策略就是轮询分配；
- **sticky 策略**：该策略在初始分配时与轮询分配类似，但是在 rebalance 时，需要保证以下两个原则：分区分配尽可能均匀；分区分配尽可能与上次分配保持相同。在两者发生冲突时，第一个原则优于第二个原则，这种策略，在 rebalance 时，基本不会出现分区全部重新分配的情况。



> #### Rebalance 过程如下

消费者加入消费者组时，消费者，消费者组和组协调器会经历以下阶段：

![](..\images\mq\kafka-rebalance.png) 



**1. 选举组协调器（GroupCoordinator）阶段**

每个消费者组都会选择一个 broker 作为组协调器，组协调器负责监控消费组中消费者心跳，判断是否宕机，开启消费者 rebalance，消费者组的消费者启动时会向 kafka 集群中某个节点发送 FindCoordinatorRequest 请求，查找对应的组协调器，并建立连接。

**组协调器的选择方式：**消费者消费的 offset 会提交到 _consumer_offsets 其中某一个分区，这个分区的 leader 副本对应的 broker 就是组协调器。



**2. 加入消费组阶段**

在成功找到对应的消费组后，进入加入消费组阶段，这个阶段消费者向组协调器发送 JoinGroupRequest 请求，并处理响应。然后组协调器选择第一个加入的消费者作为 leader，并将消费组的信息发送给 leader，由 leader 负责制定分区方案。



**3. SYNC GROUP 阶段**

consumer leader 通过给组协调器发送 SyncGroupRequest，接着组协调器就把分区方案下发给各个消费者，他们会根据指定分区的leader broker 进行网络连接以及消息消费。



> #### 生产者发布消息机制

**1. 写入方式 **

生产者采用 push 方式将消息发布到 broker（通过配置可以批量发送），每条消息都会被 append 到分区末尾，属于顺序写磁盘（顺序操作磁盘效率比随机高）。 



**2. 消息路由 **

生产者发送的消息会根据算法，将其存储到 topic 下对应的分区，路由机制为：

- 指定了分区，直接发送到对应分区；
- 未指定分区，但是指定了 key ，通过 key 的 hash 值与分区取模，得到对应分区；
- 分区和 key 都未指定，通过轮询指定分区。



**3. 写入流程 **

下面的写入流程是配置了的`acks=-1或all`情况（等待 leader 和所有 follower 写入 log 才回复生产者，允许发送下一条），而`acks=1`只需 leader 写入即可，`acks=0`在不需要等待任何副本写入 log。

- 生产者从 zk 中找到分区副本 leader；
- 将消息发送给 leader，leader 将消息写入本地 log；
- followers 从 leader pull 消息消费，写入本地 log 后向 leader 发送 ack；
- leader 收到所有 ISR 中回复的 ack 后，增加 HW（high watermark，最后 commit 的 offset），并向 producer 回复 ack；



> #### HW 和 LEO

HW俗称高水位，HighWatermark的缩写，取分区对应的**ISR中最小的LEO(log-end-offset)作为HW**，consumer最多只能消费到 HW 所在的位置。另外每个副本都有 HW，leader和follower各自负责更新自己的 HW 的状态。对于 leader 新写入的消息，consumer 不能立刻消费，leader 会等待该消息被所有 ISR 中的副本同步后更新 HW，此时消息才能被  consumer消费。这样就保证了如果 leader 所在的broker 失效，该消息仍然可以从新选举的 leader 中获取。对于来自内部 broker 的读取请求，没有HW的限制。

![](..\images\mq\kafka-hw.png)



> #### kafka在 zk 上的节点图

![](..\images\mq\kafka-zk.png)



## 四 kafka问题

### 1. 消息丢失

**消息发送端**

通过`acks`来设置 kafka 持久化方案，虽然生产者端有缓存和批次发送，但这部分数据发送失败，可以通过异步或同步发送获取发送的结果。

```properties
# acks=0；不等待任何副本写入本地log，就返回确认消息
# acks=1；leader写入本地log，返回确认消息
# acks=-1或all；等带leader和所有follows写入
spring.kafka.producer.acks=1
```



**消息消费端**

对于消费者，可以开启手动提交。若是自动提交，可能因为消费数据未处理完，就自动提交了，若此时消费者宕机，消息就无法再消费了。



### 2. 消息重复消费

**消息发送端**

发送消息时，若开启了重试机制，因为网络抖动等原因，可能导致发送端发送超时等错误，但实际 broker 已经收到消息，导致消息重复复发送。

**消费者端**

如果消费这边配置的是自动提交，刚拉取了一批数据处理了一部分，但还没来得及提交，服务挂了，下次重启又会拉取相同的一批数据重 复处理。可以在消费者端做消息幂等处理。



### 3. 消息乱序

因为分区中的消息是顺序添加，所以对于消费者端，kafka本身可以保证顺序消费。但是对于发送端，开启重试机制时，可能因为重试导致发送时消息已经乱序，也可以通过同步的方式发送，但是性能较差。



### 4. 消息积压

1. 线上有时因为发送方发送消息速度过快，或者消费方处理消息过慢，可能会导致broker积压大量未消费消息。

    此种情况如果积压了上百万未消费消息需要紧急处理，可以修改消费端程序，让其将收到的消息快速转发到其他topic(可以设置很多分 区)，然后再启动多个消费者同时消费新主题的不同分区。

2. 由于消息数据格式变动或消费者程序有bug，导致消费者一直消费不成功，也可能导致broker积压大量未消费消息。 此种情况可以将这些消费不成功的消息转发到其它队列里去(类似死信队列)，后面再慢慢分析死信队列里的消息处理问题。



### 5. 消息回溯

kafka 分区中的消息，被消费者确认后不会立即删除，而且提供了 `offsetsForTimes`、`seek`等方法，允许从某个偏移量开始重新消费。



### 6. 延时队列

kafka 中没有直接提供延时功能，但可以通过其他方式来实现。

**思路：**发送延时消息时先把消息按照不同的延迟时间段发送到指定的队列中（topic_1s，topic_5s，topic_10s，...topic_2h，这个一 般不能支持任意时间段的延时），然后通过定时器进行轮训消费这些topic，查看消息是否到期，如果到期就把这个消息发送到具体业务处 理的topic中，队列中消息越靠前的到期时间越早，具体来说就是定时器在一次消费过程中，对消息的发送时间做判断，看下是否延迟到对 应时间了，如果到了就转发，如果还没到这一次定时任务就可以提前结束了。



  







