# kafka-practice  
  
## kafka学习实践 原理分析  

- [什么是kafka](#什么是kafka)  
- [kafka产生的背景](#kafka产生的背景)  
- [kafka的应用场景](#kafka的应用场景)  
- [kafka本身的架构](#kafka本身的架构)   
- [kafka的安装部署](#kafka的安装部署)  
- [配置信息分析](#配置信息分析)  
  - [发送端的可选配置信息分析](#发送端的可选配置信息分析)  
  - [消费端的可选配置分析](#消费端的可选配置分析)  
- [关于Topic和Partition](#关于topic和partition)    
- [关于消息分发](#关于消息分发)  
- [消息的消费原理](#消息的消费原理)
- [分区分配策略](#分区分配策略)  
- [如何保存消费端的消费位置](#如何保存消费端的消费位置)  
- [#消息的存储](#消息的存储)  
  - [消息的文件存储机制](#消息的文件存储机制)  
  - [日志的清除策略以及压缩策略](#日志的清除策略以及压缩策略)  
- [partition的高可用副本机制](#partition的高可用副本机制)  
  - [数据的同步过程](#数据的同步过程)  
- [ISR的设计原理](#isr的设计原理)
  
### 什么是kafka  
  
Kafka 是一款分布式消息发布和订阅系统，具有高性能、高吞吐量的特点而被广泛应用与大数据传输场景。它是由 LinkedIn 公司开发，使用 Scala 语言编写，之后成为 Apache 基金会的一个顶级项目。kafka 提供了类似 JMS 的特性，但是在设计和实现上是完全不同的，而且他也不是 JMS 规范的实现。  

### kafka产生的背景  
  
kafka 作为一个消息系统，早期设计的目的是用作 LinkedIn 的活动流(Activity Stream)和运营数据处理管道(Pipeline)。活动流数据是所有的网站对用户的使用情况做分析的时候要用到的最常规的部分，活动数据包括页面的访问量(PV)、被查看内容方面的信息以及搜索内容。这种数据通常的处理方式是先把各种活动以日志的形式写入某种文件，然后周期性的对这些文件进行统计分析。运营数据指的是服务器的性能数据(CPU、IO 使用率、请求时间、服务日志等)。  

### kafka的应用场景  

由于 kafka 具有更好的吞吐量、内置分区、冗余及容错性的优点(kafka 每秒可以处理几十万消息)，让 kafka 成为了一个很好的大规模消息处理应用的解决方案。所以在企业级应用中，主要会应用于如下几个方面  
>行为跟踪:kafka可以用于跟踪用户浏览页面、搜索及其他行为。通过发布-订阅模式实时记录到对应的 topic 中，通过后端大数据平台接入处理分析，并做更进一步的实时处理和监控。  
>日志收集:日志收集方面，有很多比较优秀的产品，比如ApacheFlume，很多公司使用 kafka 代理日志聚合。日志聚合表示从服务器上收集日志文件，然后放到一个集中的平台(文件服务器)进行处理。在实际应用开发中，我们应用程序的 log 都会输出到本地的磁盘上，排查问题的话通过 linux 命令来搞定，如果应用程序组成了负载均衡集群，并且集群的机器有几十台以上，那么想通过日志快速定位到问题，就是很麻烦的事情了。所以一般都会做一个日志统一收集平台管理 log 日志用来快速查询重要应用的问题。  
![](https://github.com/YufeizhangRay/image/blob/master/kafka/%E6%97%A5%E5%BF%97%E6%94%B6%E9%9B%86.jpeg)  
  
### kafka本身的架构
一个典型的 kafka 集群包含若干 Producer(可以是应用节点产生的消息，也可以是通过 Flume 收集日志产生的事件)，若干个 Broker(kafka 支持水平扩展)、若干个 Consumer Group，以及一个 zookeeper 集群。kafka 通过 zookeeper 管理集群配置及服务协同。  
Producer 使用 push 模式将消息发布到 broker，consumer 通过监听使用 pull 模式从 broker 订阅并消费消息。  
多个 broker 协同工作，producer 和 consumer 部署在各个业务逻辑中。三者通过 zookeeper 管理协调请求和转发。这样就组成了一个高性能的分布式消息发布和订阅系统。  
![](https://github.com/YufeizhangRay/image/blob/master/kafka/%E6%A8%A1%E5%9E%8B.jpeg)  
图上有一个细节是和其他 mq 中间件不同的点，producer 发送消息到 broker 的过程是 push，而 consumer 从 broker 消费消息的过程是 pull，主动去拉数据。而不是 broker 把数据主动发送给 consumer。  

### kafka的安装部署  
  
下载安装包  
https://www.apache.org/dyn/closer.cgi?path=/kafka/1.1.0/kafka_2.11-1.1.0.tgz  
  
安装过程  
tar -zxvf 解压安装包即可。  
  
kafka 目录介绍  
>1./bin 操作 kafka 的可执行脚本  
>2./config 配置文件  
>3./libs 依赖库目录  
>4./logs 日志数据目录  
  
启动/停止 kafka  
>1.需要先启动 zookeeper，如果没有搭建 zookeeper 环境，可以直接运行 kafka 内嵌的 zookeeper  
启动命令: bin/zookeeper-server-start.sh config/zookeeper.properties &  
>2.进入 kafka 目录，运行 bin/kafka-server-start.sh {-daemon 后台启动} config/server.properties &  
>3.进入 kafka 目录，运行 bin/kafka-server-stop.sh config/server.properties  
  
Kafka 的基本操作  
创建 topic  
```
./kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 - -partitions 1 --topic test
```
Replication-factor 表示该 topic 需要在不同的 broker 中保存几份，这里设置 成 1，表示在两个 broker 中保存两份。  
Partitions 表示分区数。  
查看 topic  
```
./kafka-topics.sh --list --zookeeper localhost:2181  
```
查看 topic 属性  
```
./kafka-topics.sh --describe --zookeeper localhost:2181 --topic test
```
消费消息  
```
./kafka-console-consumer.sh –bootstrap-server localhost:9092 --topic test --from-beginning  
```
发送消息  
```
./kafka-console-producer.sh --broker-list localhost:9092 --topic test  
```
安装集群环境  
修改 server.properties 配置   
>1.修改 server.properties.broker.id=0 / 1  
>2.修改 server.properties 修改成本机 IP advertised.listeners=PLAINTEXT://192.168.188.138:9092  
  
当 Kafka broker 启动时，它会在 ZK 上注册自己的 IP 和端口号，客户端就通过 这个 IP 和端口号来连接。  

### 配置信息分析  
  
#### 发送端的可选配置信息分析  
  
acks  
acks 配置表示 producer 发送消息到 broker 上以后的确认值。有三个可选项 
>0:表示producer不需要等待broker的消息确认。这个选项时延最小但同时风险最大(因为当 server 宕机时，数据将会丢失)。  
>1:表示producer只需要获得kafka集群中的leader节点确认即可，这个选择时延较小同时确保了 leader 节点确认接收成功。  
>all(-1):需要ISR中所有的Replica给予接收确认，速度最慢，安全性最高，但是由于 ISR 可能会缩小到仅包含一个 Replica，所以设置参数为 all 并不能一定避免数据丢失。  
  
batch.size  
生产者发送多个消息到 broker 上的同一个分区时，为了减少网络请求带来的性能开销，通过批量的方式来提交消息，可以通过这个参数来控制批量提交的字节数大小，默认大小是 16384byte，也就是 16kb，意味着当一批消息大小达到指定的 batch.size 的时候会统一发送。  
  
linger.ms  
Producer 默认会把两次发送时间间隔内收集到的所有 Requests 进行一次聚合然后再发送，以此提高吞吐量，而 linger.ms 就是为每次发送到 broker 的请求增加一些 delay，以此来聚合更多的 Message 请求。 这个有点像 TCP 里面的 Nagle 算法，在 TCP 协议的传输中，为了减少大量小数据包的发送，采用了 Nagle 算法，也就是基于小包的等-停协议。
batch.size和linger.ms这两个参数是kafka性能优化的关键参数，这两者的作用是一样的。当二者都配置的时候，只要满足其中一个要求，就会发送请求到 broker 上。  
  
max.request.size  
设置请求的数据的最大字节数，为了防止发生较大的数据包影响到吞吐量，默认值为 1MB。  
 
#### 消费端的可选配置分析  
  
group.id  
consumer group 是 kafka 提供的可扩展且具有容错性的消费者机制。既然是一个组，那么组内必然可以有多个消费者或消费者实例(consumer instance)， 它们共享一个公共的 ID，即 group ID。组内的所有消费者协调在一起来消费订阅主题(subscribed topics)的所有分区(partition)。当然，每个分区只能由同一个消费组内的一个 consumer 来消费。如下图所示，分别有三个消费者，属于两个不同的 group，那么对于 firstTopic 这个 topic 来说，这两个组的消费者都能同时消费这个 topic 中的消息，对于此时的架构来说，这个 firstTopic 就类 似于 ActiveMQ 中的 topic 概念。
![](https://github.com/YufeizhangRay/image/blob/master/kafka/%E5%A4%9A%E7%BB%84.jpeg)  
  
如下图所示，如果 3 个消费者都属于同一个 group，那么此时 firstTopic 就是一个 Queue 的概念。  
![](https://github.com/YufeizhangRay/image/blob/master/kafka/%E5%8D%95%E7%BB%84.jpeg)  
  
enable.auto.commit  
消费者消费消息以后自动提交，只有当消息提交以后，该消息才不会被再次接收到，还可以配合 auto.commit.interval.ms 控制自动提交的频率。当然，我们也可以通过 consumer.commitSync()的方式实现手动提交。  
  
auto.offset.reset  
这个参数是针对新的 groupid 中的消费者而言的，当有新 groupid 的消费者来消费指定的 topic 时，对于该参数的配置，会有不同的语义  
>auto.offset.reset=latest 情况下，新的消费者将会从其他消费者最后消费的 offset 处开始消费 Topic 下的消息。  
>auto.offset.reset= earliest 情况下，新的消费者会从该 topic 最早的消息开始消费。  
>auto.offset.reset=none 情况下，新的消费者加入以后，由于之前不存在 offset，则会直接抛出异常。  
  
max.poll.records  
此设置限制每次调用 poll 返回的消息数，这样可以更容易的预测每次 poll 间隔要处理的最大值。通过调整此值，可以减少 poll 间隔。
  
### 关于Topic和Partition  
  
Topic  
在 kafka 中，topic 是一个存储消息的逻辑概念，可以认为是一个消息集合。每条消息发送到 kafka 集群的消息都有一个类别。物理上来说，不同的 topic 的消息是分开存储的，每个 topic 可以有多个生产者向它发送消息，也可以有多个消费者去消费其中的消息。  
![](https://github.com/YufeizhangRay/image/blob/master/kafka/%E6%8B%89%E5%8F%96.jpeg)  
  
Partition  
每个 topic 可以划分多个分区(每个 Topic 至少有一个分区)，同一 topic 下的不同分区包含的消息是不同的。每个消息在被添加到分区时，都会被分配一个 offset(称之为偏移量)，它是消息在此分区中的唯一编号，kafka 通过 offset 保证消息在分区内的顺序。offset 的顺序不跨分区，即 kafka 只保证在同一个分区内的消息是有序的。
下图中，对于名字为 test 的 topic，做了 3 个分区，分别是 p0、p1、p2。  
![](https://github.com/YufeizhangRay/image/blob/master/kafka/%E5%88%86%E7%89%87.jpeg)  
  
每一条消息发送到 broker 时，会根据 partition 的规则选择存储到哪一个 partition。如果 partition 规则设置合理，那么所有的消息会均匀的分布在不同的 partition 中， 这样就有点类似数据库的分库分表的概念，把数据做了分片处理。  

Topic&Partition 的存储  
Partition 是以文件的形式存储在文件系统中，比如创建一个名为 firstTopic 的 topic，其中有 3 个 partition，那么在 kafka 的数据目录(/tmp/kafka-log)中就有 3 个目录， firstTopic-0~3， 命名规则是<topic_name>-<partition_id> 
```
./kafka-topics.sh --create --zookeeper 192.168.188.138:2181 --replication-factor 1 --partitions 3 -- topic firstTopic  
```
    
### 关于消息分发  
  
kafka 消息分发策略  
消息是 kafka 中最基本的数据单元，在 kafka 中，一条消息由 key、value 两部分构成，在发送一条消息时，我们可以指定这个 key，那么 producer 会根据 key 和 partition 机制来判断当前这条消息应该发送并存储到哪个 partition 中。 我们可以根据需要进行扩展 producer 的 partition 机制。  
  
消息默认的分发机制  
默认情况下，kafka 采用的是 hash 取模的分区算法。如果 Key 为 null，则会随机分配一个分区。这个随机是在这个参数”metadata.max.age.ms”的时间范围内随机选择一个。对于这个时间段内，如果 key 为 null，则只会发送到唯一的分区。这个值默认情况下是 10 分钟更新一次。  
  
关于 Metadata，简单理解就是 Topic/Partition 和 broker 的映射关系，每一个 topic 的每 一个 partition，需要知道对应的 broker 列表是什么，leader 是谁、follower 是谁。这些信息都是存储在 Metadata 这个类里面。  

消费端如何消费指定的分区  
通过下面的代码，就可以消费指定该 topic 下的 0 号分区。 其他分区的数据就无法接收
```
//消费指定分区的时候，不需要再订阅 //kafkaConsumer.subscribe(Collections.singleto nList(topic));
//消费指定的分区
TopicPartition topicPartition=new TopicPartition(topic,0); kafkaConsumer.assign(Arrays.asList(topicPartit ion));
```
### 消息的消费原理  
  
kafka 消息消费原理演示  
在实际生产过程中，每个 topic 都会有多个 partitions，多个 partitions 的好处在于，一方面能够对 broker 上的数据进行分片有效减少了消息的容量从而提升 io 性能。另外一方面，为了提高消费端的消费能力，一般会通过多个 consumer 去消费同一个 topic ，也就是消费端的负载均衡机制。我们接下来要了解的，在多个 partition 以 及多个 consumer 的情况下，消费者是如何消费消息的。  
kafka 存在 consumer group 的概念，也就是 group.id 一样的 consumer，这些 consumer 属于一个 consumer group，组内的所有消费者协调在一起来消费订阅主题的所有分区。当然每一个分区只能由同一个消费组内的 consumer 来消费，那么同一个 consumer group 里面的 consumer 是怎么去分配该消费哪个分区里的数据的呢?如下图所示，3 个分区，3 个消费者，那么哪个消费者消分哪个分区?  
![](https://github.com/YufeizhangRay/image/blob/master/kafka/%E5%88%86%E7%89%87.jpeg)  
   
对于上面这个图来说，这 3 个消费者会分别消费 test 这个 topic 的 3 个分区，也就是每个 consumer 消费一个 partition。  
  
### 分区分配策略  
通过前面的案例演示，我们应该能猜到，同一个 group 中的消费者对于一个 topic 中的多个 partition，存在一定的分区分配策略。  
在 kafka 中，存在两种分区分配策略，一种是 Range(默认)、 另一种另一种还是 RoundRobin(轮询)。 通过 partition.assignment.strategy 这个参数来设置。  
  
Range strategy(范围分区)  
Range 策略是对每个主题而言的，首先对同一个主题里面 的分区按照序号进行排序，并对消费者按照字母顺序进行排序。假设我们有 10 个分区，3 个消费者，排完序的分区 将会是 0, 1, 2, 3, 4, 5, 6, 7, 8, 9；消费者线程排完序将会是 C1-0, C2-0, C3-0。然后将 partitions 的个数除于消费者线程的总数来决定每个消费者线程消费几个分区。如果除不尽，那么前面几个消费者线程将会多消费一个分区。在我们的例子里面，我们有 10 个分区，3 个消费者线程， 10 / 3=3，而且除不尽，那么消费者线程 C1-0 将会多消费一个分区，所以最后分区分配的结果看起来是这样的:
>C1-0 将消费 0, 1, 2, 3 分区  
>C2-0 将消费 4, 5, 6 分区  
>C3-0 将消费 7, 8, 9 分区  
  
假如我们有 11 个分区，那么最后分区分配的结果看起来是这样的:  
>C1-0 将消费 0, 1, 2, 3 分区  
>C2-0 将消费 4, 5, 6, 7 分区  
>C3-0 将消费 8, 9, 10 分区  
  
假如我们有 2 个主题(T1 和 T2)，分别有 10 个分区，那么最后 分区分配的结果看起来是这样的:  
>C1-0 将消费 T1主题的 0, 1, 2, 3 分区以及 T2主题的 0, 1, 2, 3 分区  
>C2-0 将消费 T1主题的 4, 5, 6 分区以及 T2主题的 4, 5, 6 分区  
>C3-0 将消费 T1主题的 7, 8, 9 分区以及 T2主题的 7, 8, 9 分区  
  
可以看出，C1-0 消费者线程比其他消费者线程多消费了 2 个分区，这就是 Range strategy 的一个很明显的弊端。  
  
RoundRobin strategy(轮询分区)  
轮询分区策略是把所有 partition 和所有 consumer 线程都列出来，然后按照 hashcode 进行排序。最后通过轮询算 法分配 partition 给消费线程。如果所有 consumer 实例的订阅是相同的，那么 partition 会均匀分布。 在我们的例子里面，假如按照 hashCode 排序完的 topic- partitions 组依次为 T1-5, T1-3, T1-0, T1-8, T1-2, T1-1, T1-4,T1-7, T1-6, T1-9，我们的消费者线程排序为 C1-0, C1-1, C2- 0, C2-1，最后分区分配的结果为:  
>C1-0 将消费 T1-5, T1-2, T1-6 分区;  
>C1-1 将消费 T1-3, T1-1, T1-9 分区;  
>C2-0 将消费 T1-0, T1-4 分区;  
>C2-1 将消费 T1-8, T1-7 分区;   
  
使用轮询分区策略必须满足两个条件  
>1.每个主题的消费者实例具有相同数量的流  
>2.每个消费者订阅的主题必须是相同的  
  
什么时候会触发这个策略呢?  
当出现以下几种情况时，kafka 会进行一次分区分配操作， 也就是 kafka consumer 的 rebalance
>1.同一个 consumer group 内新增了消费者。  
>2.消费者离开当前所属的 consumer group，比如主动停机或者宕机。  
>3.topic 新增了分区(也就是分区数量发生了变化)  
  
kafka consuemr 的 rebalance 机制规定了一个 consumer group 下的所有 consumer 如何达成一致来分配订阅 topic 的每个分区。而具体如何执行分区策略，就是前面提到过的两种内置的分区策略。而 kafka 对于分配策略这块，提供了可插拔的实现方式，也就是说，除了这两种之外，我们还可以创建自己的分配机制。  
  
谁来执行 Rebalance 以及管理 consumer 的 group 呢?  
Kafka 提供了一个角色:coordinator 来执行对于 consumer group 的管理。  
当 consumer group 的 第一个 consumer 启动的时候，它会去和 kafka server 确定谁是它们组的 coordinator。之后该 group 内的所有成员都会和该coordinator 进行协调通信。  

如何确定 coordinator  
consumer group 如何确定自己的 coordinator 是谁呢, 消费者向 kafka 集群中的任意一个 broker 发送一个 GroupCoordinatorRequest 请求，服务端会返回一个负载最小的 broker 节点的 id，并将该 broker 设置为 coordinator。  
  
JoinGroup 的过程  
在 rebalance 之前，需要保证 coordinator 是已经确定好了的，整个 rebalance 的过程分为两个步骤，Join 和 Sync join: 表示加入到 consumer group 中，在这一步中，所有的成员都会向 coordinator 发送 joinGroup 的请求。一旦所有成员都发送了 joinGroup 请求，那么 coordinator 会选择一个 consumer 担任 leader 角色，并把组成员信息和订阅信息发送消费者。
![](https://github.com/YufeizhangRay/image/blob/master/kafka/%E7%BB%93%E7%BB%84.jpeg)  
  
>protocol_metadata: 序列化后的消费者的订阅信息  
>leader_id: 消费组中的消费者，coordinator 会选择一个作为 leader   
>member_metadata: 对应消费者的订阅信息   
>members:consumer group 中全部的消费者的订阅信息   
>generation_id:年代信息，类似于之前讲解 zookeeper 的 时候的 epoch 是一样的，对于每一轮 rebalance，generation_id 都会递增。主要用来保护 。>consumer group。 隔离无效的 offset 提交。也就是上一轮的 consumer 成员无法提交 offset 到新的 consumer group 中。  
  
Synchronizing Group State 阶段  
完成分区分配之后，就进入了 Synchronizing Group State阶段，主要逻辑是向 GroupCoordinator 发送 SyncGroupRequest 请求，并且处理 SyncGroupResponse 响应，简单来说，就是 leader 将消费者对应的 partition 分配方案同步给 consumer group 中的所有 consumer。  
![](https://github.com/YufeizhangRay/image/blob/master/kafka/%E5%88%86%E6%B4%BE.jpeg)  
  
每个消费者都会向 coordinator 发送 syncgroup 请求，不过只有 leader 节点会发送分配方案，其他消费者只是打打酱油而已。当 leader 把方案发给 coordinator 以后，coordinator 会把结果设置到 SyncGroupResponse 中。这样所有成员都知道自己应该消费哪个分区。  
consumer group 的分区分配方案是在客户端执行的！Kafka 将这个权利下放给客户端主要是因为这样做可以有更好的灵活性。  
  
### 如何保存消费端的消费位置  
  
什么是 offset?  
前面在讲解 partition 的时候，提到过 offset，每个 topic可以划分多个分区(每个 Topic 至少有一个分区)，同一 topic 下的不同分区包含的消息是不同的。每个消息在被添加到分区时，都会被分配一个 offset(称之为偏移量)，它是消息在此分区中的唯一编号，kafka 通过 offset 保证消息在分区内的顺序，offset 的顺序不跨分区，即 kafka 只保证在同一个分区内的消息是有序的；对于应用层的消费来说，每次消费一个消息并且提交以后，会保存当前消费到的最近的一个 offset。  
![](https://github.com/YufeizhangRay/image/blob/master/kafka/offset.jpeg)  
  
offset 在哪里维护?  
在 kafka 中，提供了一个__consumer_offsets_* 的一个 topic，把 offset 信息写入到这个 topic 中。 
__consumer_offsets 保存了每个 consumer group 某一时刻提交的 offset 信息。  
__consumer_offsets 默认有 50 个分区。   

练习代码中，设置了一个 KafkaConsumerDemo 的 groupid。首先我们需要找到这个 consumer_group 保存在哪个分区中
```
properties.put(ConsumerConfig.GROUP_ID_CONFIG, "KafkaConsumerDemo");
```
计算公式  
```
Math.abs(“groupid”.hashCode())%groupMetadataTopicPartitionCount; 
```
由于默认情况下 groupMetadataTopicPartitionCount 有 50 个分区，计算得到的结果为:35, 意味着当前的consumer_group的位移信息保存在__consumer_offsets 的第 35 个分区。  
执行如下命令，可以查看当前 consumer_goup 中的 offset 位移信息
```
sh kafka-simple-consumer-shell.sh --topic __consumer_offsets --partition 35 --broker-list 192.168.188.138:9092,192.168.188.139:9092,192.168.188.140:9092 --formatter
```  
  
### 消息的存储  
  
消息的保存路径  
消息发送端发送消息到 broker 上以后，消息是如何持久化的呢？那么接下来去分析下消息的存储。  
首先我们需要了解的是，kafka 是使用日志文件的方式来保存生产者和发送者的消息，每条消息都有一个 offset 值来表示它在分区中的偏移量。Kafka 中存储的一般都是海量的消息数据，为了避免日志文件过大，Log 并不是直接对应在一个磁盘上的日志文件，而是对应磁盘上的一个目录，这个目录的明明规则是<topic_name>_<partition_id>   
比如创建一个名为 firstTopic 的 topic，其中有 3 个 partition， 那么在 kafka 的数据目录(/tmp/kafka-log)中就有 3 个目 录，firstTopic-0~3。  

多个分区在集群中的分配  
如果我们对于一个 topic，在集群中创建多个 partition，那 么 partition 是如何分布的呢?  
>1.将所有 N Broker 和待分配的 i 个 Partition 排序  
>2.将第 i 个 Partition 分配到第(i mod n)个 Broker 上  
  
![](https://github.com/YufeizhangRay/image/blob/master/kafka/%E5%88%86%E7%BB%84%E5%88%86%E7%89%87.jpeg)  
  
了解到这里的时候，再结合前面讲的消息分发策略，就应该能明白消息发送到 broker 上，消息会保存到哪个分区中，并且消费端应该消费哪些分区的数据了。  
  
消息写入的性能  
我们现在大部分企业仍然用的是机械结构的磁盘，如果把消息以随机的方式写入到磁盘，那么磁盘首先要做的就是寻址，也就是定位到数据所在的物理地址，在磁盘上就要找到对应的柱面、磁头以及对应的扇区；这个过程相对内存来说会消耗大量时间，为了规避随机读写带来的时间消耗，kafka 采用顺序写的方式存储数据。即使是这样，频繁的 I/O 操作仍然会造成磁盘的性能瓶颈，所以 kafka 还有一个性能策略  
  
零拷贝  
消息从发送到落地保存，broker 维护的消息日志本身就是文件目录，每个文件都是二进制保存，生产者和消费者使用相同的格式来处理。在消费者获取消息时，服务器先从 硬盘读取数据到内存，然后把内存中的数据原封不动的通过 socket 发送给消费者。虽然这个操作描述起来很简单，但实际上经历了很多步骤。
![](https://github.com/YufeizhangRay/image/blob/master/kafka/%E6%8B%B7%E8%B4%9D.jpeg)  
  
>操作系统将数据从磁盘读入到内核空间的页缓存  
>应用程序将数据从内核空间读入到用户空间缓存中  
>应用程序将数据写回到内核空间到 socket 缓存中  
>操作系统将数据从 socket 缓冲区复制到网卡缓冲区，以便将数据经网络发出  
  
这个过程涉及到 4 次上下文切换以及 4 次数据复制，并且有两次复制操作是由 CPU 完成。但是这个过程中，数据完全没有进行变化，仅仅是从磁盘复制到网卡缓冲区。  

通过“零拷贝”技术，可以去掉这些没必要的数据复制操作，同时也会减少上下文切换次数。现代的 unix 操作系统提供一个优化的代码路径，用于将数据从页缓存传输到 socket；在 Linux 中，是通过 sendfile 系统调用来完成的。Java 提供了访问这个系统调用的方法:FileChannel.transferTo  
![](https://github.com/YufeizhangRay/image/blob/master/kafka/%E9%9B%B6%E6%8B%B7%E8%B4%9D.jpeg)  
  
使用 sendfile，只需要一次拷贝就行，允许操作系统将数据直接从页缓存发送到网络上。所以在这个优化的路径中，只有最后一步将数据拷贝到网卡缓存中是需要的。  
   
#### 消息的文件存储机制  
  
前面我们知道了一个 topic 的多个 partition 在物理磁盘上的保存路径，那么我们再来分析日志的存储方式。通过如下命令找到对应 partition 下的日志内容
```
 [root@localhost ~]# ls /tmp/kafka-logs/firstTopic-1/
 ```
 ```
 00000000000000000000.index  
 00000000000000000000.log  
 00000000000000000000.timeindex  
 leader-epoch-checkpoint  
 ```
kafka 是通过分段的方式将 Log 分为多个 LogSegment， LogSegment 是一个逻辑上的概念，一个 LogSegment 对应磁盘上的一个日志文件和一个索引文件，其中日志文件是用来记录消息的。索引文件是用来保存消息的索引。  
  
LogSegment  
假设 kafka 以 partition 为最小存储单位，那么我们可以想象当 kafka producer 不断发送消息，必然会引起 partition 文件的无限扩张，这样对于消息文件的维护以及被消费的消息的清理带来非常大的挑战，所以 kafka 以 segment 为 单位又把 partition 进行细分。每个 partition 相当于一个巨型文件被平均分配到多个大小相等的 segment 数据文件中 (每个 segment 文件中的消息不一定相等)，这种特性方便已经被消费的消息的清理，提高磁盘的利用率。  
  
log.segment.bytes=107370 ( 设 置 分 段 大 小 ), 默认是 1gb，我们把这个值调小以后，可以看到日志分段的效果。
抽取其中 3 个分段来进行分析  
![](https://github.com/YufeizhangRay/image/blob/master/kafka/logsegment.jpeg)  
  
segment file 由 2 大部分组成，分别为 index file 和 data file，这 2 个文件一一对应，成对出现，后缀".index"和“.log” 分别表示为 segment 索引文件、数据文件。  
segment 文件命名规则：partion 全局的第一个 segment 从 0 开始，后续每个 segment 文件名为上一个 segment 文件最后一条消息的 offset 值进行递增。数值最大为 64 位 long 大小，20 位数字字符长度，没有数字用 0 填充。  
  
查看 segment 文件命名规则  
通过下面这条命令可以看到 kafka 消息日志的内容  
```
sh kafka-run-class.sh kafka.tools.DumpLogSegments -- files /tmp/kafka-logs/test- 0/00000000000000000000.log --print-data-log 
```
输出结果为:
```
 offset: 5376 position: 102124 CreateTime: 1531477349287
 isvalid: true keysize: -1 valuesize: 12 magic: 2
 compresscodec: NONE producerId: -1 producerEpoch: -1 
 sequence: -1 isTransactional: false headerKeys: []
 payload: message_5376
 ```
第一个 log 文件的最后一个 offset 为：5376，所以下一个 segment 的文件命名为: 00000000000000005376.log。对应的 index 为 00000000000000005376.index。  
  
segment 中 index 和 log 的对应关系  
  
从所有分段中，找一个分段进行分析  
为了提高查找消息的性能，每一个日志文件添加了 2 个索引索引文件:OffsetIndex 和 TimeIndex，分别对应*.index 以及*.timeindex，TimeIndex 索引文件格式，它是映射时间戳和相对 offset。  
查 看 索 引 内 容 : 
 ```
sh kafka-run-class.sh kafka.tools.DumpLogSegments --files /tmp/kafka- logs/test-0/00000000000000000000.index --print-data- log
 ```
 ![](https://github.com/YufeizhangRay/image/blob/master/kafka/logindex.jpeg)  
   
如图所示，index 中存储了索引以及物理偏移量。log 存储了消息的内容。索引文件的元数据执行对应数据文件中 message 的物理偏移地址。举个简单的案例来说，以 [4053,80899]为例，在 log 文件中，对应的是第 4053 条记录，物理偏移量(position)为 80899。 position 是 ByteBuffer 的指针位置。  
  
在partition中如何通过offset查找message  
  
查找的算法是  
>1.根据 offset 的值，查找 segment 段中的 index 索引文件。由于索引文件命名是以上一个文件的最后一个 offset 进行命名的，所以使用二分查找算法能够根据 offset 快速定位到指定的索引文件。  
>2.到索引文件后，根据 offset 进行定位，找到索引文件中的符合范围的索引。(kafka 采用稀疏索引的方式来提高查找性能)。  
>3.得到 position 以后，再到对应的 log 文件中，从 position 出开始查找 offset 对应的消息，将每条消息的 offset 与 目标 offset 进行比较，直到找到消息。  
  
比如说，我们要查找 offset=2490 这条消息，那么先找到 00000000000000000000.index, 然后找到[2487,49111]这个索引，再到 log 文件中，根据 49111 这个 position 开始查找，比较每条消息的 offset 是否大于等于 2490。最后查找到对应的消息以后返回。  
  
Log 文件的消息内容分析  
前面我们通过 kafka 提供的命令，可以查看二进制的日志文件信息，一条消息，会包含很多的字段。  
```
offset: 5371 position: 102124 CreateTime: 1531477349286 isvalid: true keysize: -1 valuesize: 12 magic: 2 compresscodec: NONE producerId: -1 producerEpoch: - 1 sequence: -1 isTransactional: false headerKeys: [] payload: message_5371
```
offset 和 position 这两个前面已经提过了、createTime 表示创建时间、keysize 和 valuesize 表示 key 和 value 的大小、 compresscodec 表示压缩编码、payload:表示消息的具体内容。
  
#### 日志的清除策略以及压缩策略  
  
日志清除策略  
前面提到过，日志的分段存储，一方面能够减少单个文件内容的大小，另一方面，方便 kafka 进行日志清理。日志的清理策略有两个  
>1.根据消息的保留时间，当消息在 kafka 中保存的时间超过了指定的时间，就会触发清理过程。  
>2.根据 topic 存储的数据大小，当 topic 所占的日志文件大小大于一定的阈值，则可以开始删除最旧的消息。  
  
kafka 会启动一个后台线程，定期检查是否存在可以删除的消息。通过 log.retention.bytes 和 log.retention.hours 这两个参数来设置，当其中任意一个达到要求，都会执行删除。默认的保留时间是:7天  
  
日志压缩策略  
Kafka 还提供了“日志压缩(Log Compaction)”功能，通过这个功能可以有效的减少日志文件的大小，缓解磁盘紧张的情况，在很多实际场景中，消息的 key 和 value 的值之间的对应关系是不断变化的，就像数据库中的数据会不断被修改一样，消费者只关心 key 对应的最新的 value。因此，我们可以开启 kafka 的日志压缩功能，服务端会在后台启动启动 Cleaner 线程池，定期将相同的 key 进行合并，只保留最新的 value 值。日志的压缩原理是  
![](https://github.com/YufeizhangRay/image/blob/master/kafka/%E6%97%A5%E5%BF%97%E5%8E%8B%E7%BC%A9.jpeg)  
  
### partition的高可用副本机制  
  
我们已经知道 Kafka 的每个 topic 都可以分为多个 partition， 并且多个 partition 会均匀分布在集群的各个节点下。虽然这种方式能够有效的对数据进行分片，但是对于每个 partition 来说，都是单点的，当其中一个 partition 不可用的时候，那么这部分消息就没办法消费。所以 kafka 为了提高 partition 的可靠性而提供了副本的概念(Replica)，通过副本机制来实现冗余备份。  
每个分区可以有多个副本，并且在副本集合中会存在一个 leader 的副本，所有的读写请求都是由 leader 副本来进行处理。剩余的其他副本都做为 follower 副本，follower 副本会从 leader 副本同步消息日志。这个有点类似 zookeeper 中 leader 和 follower 的概念，但是具体的时间方式还是有比较大的差异。所以我们可以认为，副本集会存在一主多从的关系。  
一般情况下，同一个分区的多个副本会被均匀分配到集群中的不同 broker 上，当 leader 副本所在的 broker 出现故障后，可以重新选举新的 leader 副本继续对外提供服务。通过这样的副本机制来提高 kafka 集群的可用性。  
  
副本分配算法  
>将所有 N Broker 和待分配的 i 个 Partition 排序。  
>将第 i 个 Partition 分配到第(i mod n)个 Broker 上。  
>将第 i 个 Partition 的第 j 个副本分配到第((i + j) mod n)个 Broker 上。  
  
通过下面的命令去创建带 2 个副本的 topic  
```
 ./kafka-topics.sh --create --zookeeper 192.168.188.138:2181 --replication-factor 2 --partitions 3 -- topic secondTopic
 ```
然后我们可以在/tmp/kafka-log 路径下看到对应 topic 的 副本信息了。我们通过一个图形的方式来表达。 
针对 secondTopic 这个 topic 的 3 个分区对应的 3 个副本
![](https://github.com/YufeizhangRay/image/blob/master/kafka/%E5%A4%87%E4%BB%BD.jpeg)  
  
如何知道那个各个分区中对应的 leader 是谁呢?  
在 zookeeper 服务器上，通过如下命令去获取对应分区的 信息，比如下面这个是获取 secondTopic 第 1 个分区的状态信息。  
```
get /brokers/topics/secondTopic/partitions/1/state
```
```
{"controller_epoch":12,"leader":0,"version":1,"leader_epoch":0,"isr":[0,1]}
```
leader 表示当前分区的 leader 是那个 broker-id。下图中绿色线条的表示该分区中的 leader 节点。其他节点就为 follower  
![](https://github.com/YufeizhangRay/image/blob/master/kafka/%E5%A4%87%E4%BB%BD%E4%B8%BB%E4%BB%8E.jpeg)  
  
Kafka 提供了数据复制算法保证，如果 leader 发生故障或挂掉，一个新 leader 被选举并被接受客户端的消息成功写入。Kafka 确保从同步副本列表中选举一个副本为 leader；leader 负责维护和跟踪 ISR(in-Sync replicas，副本同步队列)中所有 follower 滞后的状态。当 producer 发送一条消息到 broker 后，leader 写入消息并复制到所有 follower。消息提交之后才被成功复制到所有的同步副本。  
需要注意的是，不能把 zookeeper 的 leader 和 follower 的同步机制和 kafka 副本的同步机制搞混了。虽然从思想层面来说是一样的，但是原理层面的实现是完全不同的。
  
kafka 副本机制中的几个概念  
Kafka 分区下有可能有很多个副本(replica)用于实现冗余，从而进一步实现高可用。副本根据角色的不同可分为 3 类:   
>leader 副本:响应 clients 端读写请求的副本。  
>follower 副本:被动地备份 leader 副本中的数据，不能响应 clients 端读写请求。  
>ISR 副本:包含了 leader 副本和所有与 leader 副本保持同步的 follower 副本。  
  
每个 Kafka 副本对象都有两个重要的属性:LEO 和 HW。注意是所有的副本，而不只是 leader 副本。  
>LEO:即日志末端位移(log end offset)，记录了该副本底层日志(log)中下一条消息的位移值。注意是下一条消息!也就是说，如果 LEO=10，那么表示该副本保存了 10 条消息，位移值范围是[0, 9]。另外，leader LEO 和 follower LEO 的更新是有区别的。  
>HW:即上面提到的水位值。对于同一个副本对象而言，其 HW 值不会大于 LEO 值。小于等于 HW 值的所有消息都被 认为是“已备份”的(replicated)。同理，leader 副本和 follower 副本的 HW 更新是有区别的。  
  
副本协同机制  
刚刚提到了，消息的读写操作都只会由 leader 节点来接收和处理。follower 副本只负责同步数据以及当 leader 副本所在的 broker 挂了以后，会从 follower 副本中选取新的 leader。  
![](https://github.com/YufeizhangRay/image/blob/master/kafka/%E5%88%86%E7%89%87%E5%90%8C%E6%AD%A5.jpeg)  
  
写请求首先由 leader 副本处理，之后 follower 副本会从 leader 上拉取写入的消息，这个过程会有一定的延迟，导致 follower 副本中保存的消息略少于 leader 副本，只要没有超出阈值都可以容忍。但是如果一个 follower 副本出现异常，比如宕机、网络断开等原因长时间没有同步到消息，那这个时候，leader 就会把它踢出去。kafka 通过 ISR 集合来维护一个分区副本信息。  
  
ISR  
ISR 表示目前“可用且消息量与 leader 相差不多的副本集合，这是整个副本集合的一个子集”。怎么去理解可用和相差不多这两个词呢？具体来说，ISR 集合中的副本必须满足两个条件  
>1.副本所在节点必须维持着与 zookeeper 的连接。  
>2.副本最后一条消息的 offset 与 leader 副本的最后一条消息 的 offset 之间的差值不能超过指定的阈值 (replica.lag.time.max.ms)。  
  
replica.lag.time.max.ms:如果该 follower 在此时间间隔内一直没有追上过 leader 的所有消息，则该 follower 就会被剔除 isr 列表。  
  
HW&LEO  
关于 follower 副本同步的过程中，还有两个关键的概念，HW(HighWatermark)和 LEO(Log End Offset)。这两个参数跟 ISR 集合紧密关联。HW 标记了一个特殊的 offset，当消费者处理消息的时候，只能拉取到 HW 之前的消息，HW 之后的消息对消费者来说是不可见的。也就是说，取 partition 对应 ISR 中最小的 LEO 作为 HW，consumer 最多只能消费到 HW 所在的位置。每个 replica 都有 HW， leader 和 follower 各自维护更新自己的 HW 的状态。一条消息只有被 ISR 里的所有 follower 都从 leader 复制过去才会被认为已提交。这样就避免了部分数据被写进了 leader，还没来得及被任何 follower 复制就宕机了，而造成数据丢失(Consumer 无法消费这些数据)。而对于 Producer 而言，它可以选择是否等待消息 commit，这可以通过 acks 来设置。这种机制确保了只要 ISR 有一个或以上的 follower，一条被 commit 的消息就不会丢失。  

#### 数据的同步过程  
了解了副本的协同过程以后，还有一个最重要的机制，就是数据的同步过程。它需要解决  
>1.怎么传播消息  
>2.在向消息发送端返回 ack 之前需要保证多少个 Replica 已经接收到这个消息  
  
数据的处理过程是Producer 在发布消息到某个 Partition 时，先通过ZooKeeper 找到该 Partition 的 leader，然后无论该 Topic 的 Replication Factor 为多少(也即该 Partition 有多少个 Replica)，Producer 只将该消息发送到该 Partition 的 leader。leader 会将该消息写入其本地 Log。每个 follower 都从 leader pull 数据。这种方式上，follower 存储的数据顺序与 leader 保持一致。follower 在收到该消息并写入其 Log 后，向 leader 发送 ACK。一旦 leader 收到了 ISR 中的所有 Replica 的 ACK，该消息就被认为已经 commit 了，leader 将增加 HW(HighWatermark)并且向 Producer 发送 ACK。  
  
初始状态  
初始状态下，leader 和 follower 的 HW 和 LEO 都是 0，leader 副本会保存 remote LEO，表示所有 follower LEO，也会被初始化为 0，这个时候，producer 没有发送消息。follower 会不断地个 leader 发送 FETCH 请求，但是因为没有数据，这个请求会被 leader 寄存，当在指定的时间之后会强制完成请求，这个时间配置是 (replica.fetch.wait.max.ms)，如果在指定时间内 producer 有消息发送过来，那么 kafka 会唤醒 fetch 请求，让 leader 继续处理。  
![](https://github.com/YufeizhangRay/image/blob/master/kafka/%E5%90%8C%E6%AD%A51.jpeg)  
  
这里会分两种情况，第一种是 leader 处理完 producer 请求之后，follower 发送一个 fetch 请求过来、第二种是 follower 阻塞在 leader 指定时间之内，leader 副本收到 producer 的请求。这两种情况下处理方式是不一样的。先来看第一种情况  
follower 的 fetch 请求是当 leader 处理消息以后执行的  
生产者发送一条消息  
leader 处理完 producer 请求之后，follower 发送一个 fetch 请求过来 。状态图如下  
![](https://github.com/YufeizhangRay/image/blob/master/kafka/%E5%90%8C%E6%AD%A52.jpeg)  
  
leader 副本收到请求以后，会做几件事情  
>1.把消息追加到 log 文件，同时更新 leader 副本的 LEO  
>2.尝试更新 leader HW 值。这个时候由于 follower 副本还没有发送 fetch 请求，那么 leader 的 remote LEO 仍然是 0。leader 会比较自己的 LEO 以及 remote LEO 的值，发现最小值是 0，与 HW 的值相同，所以不会更新 HW。  
  
follower fetch 消息  
![](https://github.com/YufeizhangRay/image/blob/master/kafka/%E5%90%8C%E6%AD%A53.jpeg)  
  
follower 发送 fetch 请求，leader 副本的处理逻辑是:  
>1.读取 log 数据、更新 remote LEO=0(follower 还没有写 入这条消息，这个值是根据 follower 的 fetch 请求中的 offset 来确定的)。  
>2.尝试更新 HW，因为这个时候 LEO 和 remoteLEO 还是不一致，所以仍然是 HW=0。  
>3. 把消息内容和当前分区的 HW 值发送给 follower 副本  
>follower 副本收到 response 以后  
>>a)将消息写入到本地 log，同时更新 follower 的 LEO。  
>>b)更新 follower HW，本地的 LEO 和 leader 返回的 HW 进行比较取小的值，所以仍然是 0。  
  
第一次交互结束以后，HW 仍然还是 0，这个值会在下一次 follower 发起 fetch 请求时被更新  
![](https://github.com/YufeizhangRay/image/blob/master/kafka/%E5%90%8C%E6%AD%A54.jpeg)  
  
follower 发第二次 fetch 请求，leader 收到请求以后
>1.读取 log 数据  
>2.更新 remote LEO=1， 因为这次 fetch 携带的 offset 是 1。  
>3.更新当前分区的 HW，这个时候 leader LEO 和 remote LEO 都是 1，所以 HW 的值也更新为 1。  
>4.把数据和当前分区的 HW 值返回给 follower 副本，这个时候如果没有数据，则返回为空。  
  
follower 副本收到 response 以后  
>1.如果有数据则写本地日志，并且更新 LEO  
2.更新 follower 的 HW 值  
  
到目前为止，数据的同步就完成了，意味着消费端能够消 费 offset=0 这条消息。  

follower 的 fetch 请求是直接从阻塞过程中触发  
前面说过，由于 leader 副本暂时没有数据过来，所以 follower 的 fetch 会被阻塞，直到等待超时或者 leader 接收到新的数据。当 leader 收到请求以后会唤醒处于阻塞的 fetch 请求。处理过程基本上和前面说的一致
>1.leader 将消息写入本地日志，更新 Leader 的 LEO。  
>2.唤醒 follower 的 fetch 请求。  
>3.更新 HW。  
  
kafka 使用 HW 和 LEO 的方式来实现副本数据的同步，本身是一个好的设计，但是因为 HW 的值是在新的一轮 FETCH 中才会被更新，在这个地方会存在一个数据丢失的问题。当然这个丢失只出现在特定的背景下，我们来分析下这个过程为什么会出现数据丢失。  
  
数据丢失的问题  
前提:min.insync.replicas=1 的时候。设定 ISR 中的最小副本数是多少，默认值为 1, 当且仅当 acks 参数设置为-1 (表示需要所有副本确认)时，此参数才生效。表达的含义是，至少需要多少个副本同步才能表示消息是提交的所以，当 min.insync.replicas=1 的时候，一旦消息被写入 leader 端 log 即被认为是“已提交”，而延迟一轮 FETCH RPC 更新 HW 值的设计使得 follower HW 值是异步延迟更新的，倘若在这个过程中 leader 发生变更，那么成为新 leader 的 follower 的 HW 值就有可能是过期的，使得 clients 端认为是成功提交的消息被删除。  
![](https://github.com/YufeizhangRay/image/blob/master/kafka/%E6%95%B0%E6%8D%AE%E9%98%B2%E4%B8%A2%E5%A4%B1.jpeg)  
  
数据丢失的解决方案  
在 kafka0.11.0.0 版本以后，提供了一个新的解决方案，使用 leader epoch 来解决这个问题，leader epoch 实际上是一对(epoch,offset), epoch 表示 leader 的版本号，从 0 开始，当 leader 变更过 1 次时 epoch 就会+1，而 offset 则对应于该 epoch 版本的 leader 写入第一条消息的位移。  
比如说(0,0)；(1,50)；表示第一个 leader 从 offset=0 开始写消息，一共写了 50 条，第二个 leader 版本号是 1，从 50 条处开始写消息。这个信息保存在对应分区的本地磁盘文件中，文件名为 : /tml/kafka-log/topic/leader-epoch-checkpoint。leader broker 中会保存这样的一个缓存，并定期地写入到一个 checkpoint 文件中。  
当 leader 写 log 时它会尝试更新整个缓存，如果这个 leader 首次写消息，则会在缓存中增加一个条目，否则就不做更新。而每次副本重新成为 leader 时会查询这部分缓存，获取出对应 leader 版本的 offset。
   
如何处理所有的 Replica 不工作的情况  
在 ISR 中至少有一个 follower 时，Kafka 可以确保已经 commit 的数据不丢失，但如果某个 Partition 的所有 Replica 都宕机了，就无法保证数据不丢失了
>1.等待 ISR 中的任一个 Replica“活”过来，并且选它作为Leader。  
>2.选择第一个“活”过来的 Replica(不一定是 ISR 中的)作为 Leader。  
  
这就需要在可用性和一致性当中作出一个简单的折衷。如果一定要等待 ISR 中的 Replica“活”过来，那不可用的时间就可能会相对较长。而且如果 ISR 中的所有 Replica 都无法“活”过来了，或者数据都丢失了，这个 Partition 将永远不可用。  
选择第一个“活”过来的 Replica 作为 Leader，而这个 Replica 不是 ISR 中的 Replica，那即使它并不保证已经包含了所有已 commit 的消息，它也会成为 Leader 而作为 consumer 的数据源。
  
### ISR的设计原理  
在所有的分布式存储中，冗余备份是一种常见的设计方式，而常用的模式有同步复制和异步复制，按照 kafka 这个副本模型来说，如果采用同步复制，那么需要要求所有能工作的 Follower 副本都复制完，这条消息才会被认为提交成功，一旦有一个 follower 副本出现故障，就会导致 HW 无法完成递增，消息就无法提交，消费者就获取不到消息。这种情况下，故障的 Follower 副本会拖慢整个系统的性能，设置导致系统不可用如果采用异步复制，leader 副本收到生产者推送的消息后，就认为次消息提交成功。follower 副本则异步从 leader 副本同步。这种设计虽然避免了同步复制的问题，但是假设所有 follower 副本的同步速度都比较慢他们保存的消息量远远落后于 leader 副本。而此时 leader 副本所在的 broker 突然宕机，则会重新选举新的 leader 副本，而新的 leader 副本中没有原来 leader 副本的消息。这就出现了消息的丢失。  
kafka 权衡了同步和异步的两种策略，采用 ISR 集合，巧妙解 决了两种方案的缺陷:当 follower 副本延迟过高，leader 副本则会把该 follower 副本提出 ISR 集合，消息依然可以快速提交。当 leader 副本所在的 broker 突然宕机，会优先将 ISR 集合中 follower 副本选举为 leader，新 leader 副本包含了 HW 之前 的全部消息，这样就避免了消息的丢失。
 
