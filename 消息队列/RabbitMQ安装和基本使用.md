

## 1. 安装

### 1.1 安装erlang语言环境

```sh
# 直接下载安装包
https://packagecloud.io/rabbitmq/erlang/install
# bash-rpm
wget https://packagecloud.io/rabbitmq/erlang/packages/el/7/erlang-23.2.1-1.el7.x86_64.rpm
# 用命令安装
curl -s https://packagecloud.io/install/repositories/rabbitmq/erlang/script.rpm.sh | sudo bash

# 安装erlang
yum install -y erlang
```



### 1.2 安装socat加密软件

```sh
yum -y install epel-release
yum -y install socat
```



### 1.3 安装mq

```sh
# 下载安装包
https://github.com/rabbitmq/rabbitmq-server/
yum install -y xxx.rpm

# 安装管理平台插件
rabbitmq-plugins enable rabbitmq_management

# 启动mq
systemctl start rabbitmq-server
```



## 2. AMQP协议

**AMQP协议是一个应用层的协议规范。**

**AMQP协议模型**

![](..\images\mq\amqp.png)

**AMQP模型几个核心概念：**

- **Message Queue**：消息队列；用作存储消息的缓冲区；
- **Exchanges**：交换器；生产者的消息会被投递到交换器上，然后又交换器根据路由 key 发送到对应的队列上。
- **Binding**：绑定；交换器和队列的虚拟绑定，可以通过路由 key 来绑定指定的队列；
- **Messages**：消息；生产者和消费者之间传递的数据信息，包括 properties(消息属性，例如优先级、存活时间等)和 body(消息体)；
- **Connection**：连接；程序和 broker 之间的连接，例如 TCP；
- **Channel**：通道；是连接内部的虚拟连接，用于向队列投递消息或者消费消息，是多路复用的；
- **Virtual Hosts**：虚拟机；一个逻辑上的概念，包含多个交换机和队列。



## 3. RabbitMQ工作模式

### 1. Work Queues

**当没有指定交换机时，MQ 会提供默认的交换机。**

![image-20210110182832098](..\images\mq\work.png)  



示例：

```java
public class RabbitMQUitl {

    public static Connection getConnection() {
        ConnectionFactory factory = new ConnectionFactory();
        // ip
        factory.setHost("192.168.0.100");
        // 端口
        factory.setPort(5672);
        // 虚拟机
        factory.setVirtualHost("my_vhost");
        factory.setUsername("mq");
        factory.setPassword("mq");
        factory.setConnectionTimeout(10000);
        
        Connection connection = null;
        try {
            connection = factory.newConnection();
        } catch (Exception e) {
            e.printStackTrace();
        }
        return connection;
    }
}
```

**消费者：**

```java
public class SimpaleConsumer {

    public static void main(String[] args) throws Exception {
        Connection connection = RabbitMQUitl.getConnection();
        Channel channel = connection.createChannel();
        String queueName = "simpaleQuene";
        /**
         * 声明队列
         * queue: 队列
         * durable：是否持久化，true:是；false:否
         * exclusive：是否是独占队列，其他消费者是否可以访问，true:是；false:否
         * autoDelete：消费完之后是否自动删除
         * arguments：其他一些结构化的参数
         *
         */
        channel.queueDeclare("simpaleQuene", true, false, true, null);

        // 消费消息
        channel.basicConsume("simpaleQuene", new DefaultConsumer(channel){
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                System.out.println(new String(body));
            }
        });
    }
}
```

**生产者：**

```java
public class SimpaleProducer {

    public static void main(String[] args) throws Exception {
        Connection connection = RabbitMQUitl.getConnection();
        Channel channel = connection.createChannel();
        for (int i = 0; i < 10; i++) {
            String message = "this is " + i + " message";
            channel.basicPublish("","simpaleQuene", null, message.getBytes());
        }
        channel.close();
        connection.close();
    }
}
```



### 2. Publish/Subscribe

这种工作模式需要将交换机的类型设置为`fanout`。

![](..\images\mq\publish.png) 

**消费者**

```java
public class FanoutConsumer {

    public static void main(String[] args) throws Exception {
        Connection connection = RabbitMQUitl.getConnection();
        Channel channel = connection.createChannel();
        // 创建一个非持久、排他、自动删除的队列
        String queue = channel.queueDeclare().getQueue();
        // 绑定队列和交换机
        channel.queueBind(queue, "fanout_exchanges","");

        channel.basicConsume(queue, new DefaultConsumer(channel){
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                System.out.println(new String(body));
            }
        });
    }
}
```

**生产者**

```java
public class FanoutProducer {

    public static void main(String[] args) throws Exception {
        Connection connection = RabbitMQUitl.getConnection();
        Channel channel = connection.createChannel();
        /**
         * 声明创建交换机 
         * exchange：交换机名称
         * type：类型；fanout：扇形交换机
         */
        channel.exchangeDeclare("fanout_exchanges", "fanout");

        String message = "this is fanout message";
        channel.basicPublish("fanout_exchanges","",null, message.getBytes());

        channel.close();
        connection.close();
    }
}
```



### 3. Routing

设置交换机的类型为`direct`，并通过路由 key 与队列绑定。

![](..\images\mq\routing.png) 

**消费者：**

```java
public class FanoutConsumer {

    public static void main(String[] args) throws Exception {
        Connection connection = RabbitMQUitl.getConnection();
        Channel channel = connection.createChannel();
        // 创建一个非持久、排他、自动删除的队列
        String queue = channel.queueDeclare().getQueue();
        // 绑定队列和交换机
        channel.queueBind(queue, "fanout_exchanges","");

        while(true) {
            channel.basicConsume(queue, new DefaultConsumer(channel){
                @Override
                public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                    System.out.println(new String(body));
                }
            });
        }
    }
}
```

**生产者：**

```java
public class FanoutProducer {

    public static void main(String[] args) throws Exception {
        Connection connection = RabbitMQUitl.getConnection();
        Channel channel = connection.createChannel();
        /**
         * exchange：交换机名称
         * type：类型；fanout：扇形交换机
         */
        channel.exchangeDeclare("fanout_exchanges", "fanout");

        String message = "this is fanout message";
        channel.basicPublish("fanout_exchanges","",null, message.getBytes());

        channel.close();
        connection.close();
    }
}
```



### 4.  Topics

设置交换机的类型为`direct`，并通过路由 key 与队列绑定。

- `*`：匹配一个；
- `#`：匹配0个或多个。

![](..\images\mq\topic.png) 

**消费者：**

```java
public class TopicConsumer {

    public static void main(String[] args) throws Exception{
        Connection connection = RabbitMQUitl.getConnection();
        Channel channel = connection.createChannel();
        channel.queueDeclare("topic1", false, false, false,null);
        channel.queueBind("topic1", "topic_exchanges","topic.*");
       while (true) {
           channel.basicConsume("topic1", true, new DefaultConsumer(channel){
               @Override
               public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                   System.out.println(new String(body));
               }
           });
       }
    }
}

public class TopicConsumer2 {

    public static void main(String[] args) throws Exception{
        Connection connection = RabbitMQUitl.getConnection();
        Channel channel = connection.createChannel();
        channel.queueDeclare("topic2", false, false, false,null);
        channel.queueBind("topic2", "topic_exchanges","topic.#");
        while (true) {
            channel.basicConsume("topic2", true, new DefaultConsumer(channel){
                @Override
                public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                    System.out.println(new String(body));
                }
            });
        }
    }
}
```

**生产者：**

```java
public class TopicProducer {

    public static void main(String[] args) throws Exception{
        Connection connection = RabbitMQUitl.getConnection();
        Channel channel = connection.createChannel();
        channel.exchangeDeclare("topic_exchanges","topic",true);
        String message = "topic.message";
        String message2 = "topic.message.context";
        for (int i = 0; i < 10; i++) {
            channel.basicPublish("topic_exchanges", "topic",null,message.getBytes());
            channel.basicPublish("topic_exchanges", "topic.message.context",null,message2.getBytes());
        }
        channel.close();
        connection.close();
    }
}
```





## 4. 镜像模式

在镜像集群模式下，你创建的 queue，无论元数据还是 queue 里的消息都会**存在于多个实例上**，就是说，每个 RabbitMQ 节点都有这个 queue 的一个**完整镜像**，包含 queue 的全部数据的意思。然后每次你写消息到 queue 的时候，都会自动把**消息同步**到多个实例的 queue 上。

**搭建步骤：**

1. 把主节点的 cookie 复制到从节点上：

```sh
scp /var/lib/rabbitmq/.erlang.cookie 172.16.32.18:/var/lib/rabbitmq/
```

