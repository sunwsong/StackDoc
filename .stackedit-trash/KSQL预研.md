# KSQL

## Apache Kafka

Apache Kafka是一个分布式流媒体平台，它有三个关键功能：

- 能够发布和订阅。
- 容错，可以持久存储数据。
- 延时低。

### 概念

- Kafka作为一个集群，运行在一个或多个数据中心的服务器上。
- Kafka集群以Topic存储消息流。
- 每条消息都包含一个键，一个值和一个时间戳

#### Topic

topic是发布消息和订阅消息的名称，一个topic可以有多个用户发布数据或消费数据。

对于每个主题，kafka集群都维护一个如下所示的分区日志：
![](https://kafka.apache.org/11/images/log_anatomy.png)
每个分区都是一个有序的、不可变的消息序列。分区中的每条记录都被分配了一个递增的offset，它唯一标识了分区中的每条记录。

#### 容错

日志的分区分布在Kafka集群的服务器上。每个分区都可以复制到多个服务器上，以实现容错。

每个分区都有一个服务器充当leader，零个或多个服务器充当follower。leader负责处理分区所有读取和写入的请求，而follower同步复制leader上的数据。如果leader出现错误，无法正常工作，其中的一个follower会自动成为新的leader。每个服务其都充当着某些分区的leader和其它分区的follower。

#### 生产者

生产者将数据发布到指定的Topic。

#### 消费者

消费者用group_id标记自己。Topic中的记录会被传递到订阅该topic的消费者组中的一个消费者实例。消费者示例可以在单独的进程中，也可以在不同的机器上。

如果消费者实例具有相同的group_id，则kafka会有效的进行负载均衡，如下：
![](https://kafka.apache.org/11/images/consumer-groups.png)

### 应用

#### 作为消息系统

#### 作为存储系统

## Kafka Streams

Kafka Streams通过构建Kafka生产者和消费者API，简化程序开发，并利用Kafka本身的功能提供数据的并行性、分布式协调、容错和操作简便性。

以下是使用Kafka Streams API的应用程序的解剖结构。它提供了包含多个流程的Kafka Streams应用程序的逻辑视图，每个线程包含多个流任务。
![../_images/streams-architecture-overview.jpg](file:///D:/idea.png)

### Processor Topology

Processor Topology用来描述输入数据如何被转换成输出数据的处理逻辑。拓扑（Topology）通过流（Streams）连接的流处理器（Processor）形成图形。拓扑中有两个特殊的处理器：

- Source Processor：从一个或多个Kafka Topic为拓扑生成输入流。
- Sink Processor：将从上游处理器接收到的记录发送到指定Kafka Topic。

![../_images/streams-architecture-topology.jpg](https://docs.confluent.io/current/_images/streams-architecture-topology.jpg)

### 并行模型

#### Partitions和Tasks

Kafka通过分区来存储和传输数据。Kafka Streams也通过分区对数据进行处理。这两种情况下，分区都可以实现数据的局部性，弹性，可伸缩性，高性能和容错性。

Kafka Streams使用Stream Partitions和Stream Tasks作为其并行模型的逻辑单元。这与Kafka之间有着密切的联系：

- 每个Stream Partition 都是完全有序的数据序列，对应到一个Kafka Topic的分区。
- 一个Kafka Stream对应一个Kafka Topic

#### 线程模型

Kafka Streams允许用户配置用于并行化处理的线程数。每个线程可以独立执行一个或多个Stream Tasks。![../_images/streams-architecture-threads.jpg](https://docs.confluent.io/current/_images/streams-architecture-threads.jpg)

启动更多的流处理线程仅仅是复制拓扑，并使其处理Kafka分区的不同子集，从而有效地并行处理。值得注意的是，线程之间没有共享状态，因此不需要线程间协调。Kafka Streams利用Kafka服务器端的协调功能透明的处理各种流线程中Kafka分配的主题分区。![../_images/streams-architecture-example-02.png](https://docs.confluent.io/current/_images/streams-architecture-example-02.png)

### State

Kafka Streams提供状态存储，Streams处理程序可以使用它来存储和查询数据，这是实现有状态存储的一项重要功能。

<!--stackedit_data:
eyJoaXN0b3J5IjpbLTE2MTk3OTQ4MjJdfQ==
-->