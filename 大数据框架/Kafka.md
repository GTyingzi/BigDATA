&emsp;<a href="#0">Kafka</a>  
&emsp;&emsp;<a href="#1">Kafka的概念</a>  
&emsp;&emsp;&emsp;<a href="#2">Kafka的特点、优缺点</a>  
&emsp;&emsp;&emsp;<a href="#3">Kafka的使用场景</a>  
&emsp;&emsp;&emsp;<a href="#4">Kafka架构</a>  
&emsp;&emsp;<a href="#5">Kafka的生产者区域</a>  
&emsp;&emsp;&emsp;<a href="#6">分区策略</a>  
&emsp;&emsp;&emsp;<a href="#7">数据可靠性保证</a>  
&emsp;&emsp;&emsp;&emsp;<a href="#8">1）副本数据同步策略</a>  
&emsp;&emsp;&emsp;&emsp;<a href="#9">2）ISR机制</a>  
&emsp;&emsp;&emsp;&emsp;<a href="#10">3）ack应答机制</a>  
&emsp;&emsp;&emsp;&emsp;<a href="#11">4）故障处理细节</a>  
&emsp;&emsp;&emsp;<a href="#12">Exactly Onec语义</a>  
&emsp;&emsp;&emsp;<a href="#13">生产者发送的一条 message 中包含哪些信息</a>  
&emsp;&emsp;&emsp;<a href="#14">生产者向Kafka发送消息的执行流程</a>  
&emsp;&emsp;&emsp;<a href="#15">kafka文件存储机制</a>  
&emsp;&emsp;<a href="#16">Kafka的消费者区域</a>  
&emsp;&emsp;&emsp;<a href="#17">消费方式</a>  
&emsp;&emsp;&emsp;<a href="#18">分区分配策略</a>  
&emsp;&emsp;&emsp;<a href="#19">kafka的消费者组跟分区之间的关系</a>  
&emsp;&emsp;&emsp;<a href="#20">offset的维护</a>  
&emsp;&emsp;&emsp;<a href="#21">如何实现 kafka 消费者每次只消费指定数量的消息</a>  
&emsp;&emsp;&emsp;<a href="#22">kafka如何实现多线程的消费</a>  
&emsp;&emsp;&emsp;<a href="#23">kafka消费支持几种消费模式</a>  
&emsp;&emsp;<a href="#24">综合</a>  
&emsp;&emsp;<a href="#25">Kafka高效读写数据</a>  
&emsp;&emsp;&emsp;<a href="#26">Zookeeper在Kafka中的作用</a>  
&emsp;&emsp;&emsp;<a href="#27">Kafka事务</a>  
&emsp;&emsp;&emsp;<a href="#28">kafka如何实现消息是有序的</a>  
&emsp;&emsp;&emsp;<a href="#29">kafka的分区算法</a>  
&emsp;&emsp;&emsp;<a href="#30">kafka的默认消息保留策略</a>  
&emsp;&emsp;&emsp;<a href="#31">kafka如何实现单个集群间的消息复制</a>  
&emsp;&emsp;&emsp;<a href="#32">LEO、HW、LSO、LW分别代表什么</a>  
&emsp;&emsp;&emsp;<a href="#33">如何保证每个应用程序都可以获取到 Kafka 主题中的所有消息，而不是部分消息</a>  
&emsp;&emsp;&emsp;<a href="#34">Kafka的选举机制</a>  
&emsp;&emsp;&emsp;<a href="#35">kafka如何清理过期数据</a>  
<img src="https://yingziimage.oss-cn-beijing.aliyuncs.com/img/image-20220409112003412.png" alt="image-20220409112003412" style="zoom:200%;" />

## <a name="0">Kafka


### <a name="1">Kafka的概念


Kafka是一种分布式、高吞吐量的**分布式分布订阅消息系统**，它可以处理消费者规模的网站中的所有动作流数据，主要应用于大数据实时处理领域。

类比来说，kafka是一个邮箱，生产者是发送邮件的人，消费者是接收邮件的人，Kafka是用来存东西的，只不过它提供了一些处理邮件的机制。

#### <a name="2">Kafka的特点、优缺点


特点

````
高吞吐量、低延迟：每秒可以处理几十万条消息，它的延迟最低只有几毫米

可扩展性：kafka集群支持热扩展

持久性、可靠性：消息被持久化到本地磁盘，并且支持数据备份防止数据丢失

容错性：运行集群中节点故障（若副本数量为n，则允许n-1个节点故障）

高并发：支持数千个客户端同时读写
````

优点：

```
支持多个生产者和消费者
支持broker的横向拓展
副本集机制，实现数据冗余，保证数据不丢失
通过topic将数据进行分类
通过分批发送压缩数据的方式，减少数据传输开销，提高吞吐量
支持多种模式的消息
基于磁盘实现数据的持久化
高性能的处理消息，在大数据的情况下，可以保证亚秒级的消息延迟
一个消费者可以支持多种topic的消息
对CPU和内存的消耗比较小
对网络开销也比较小
支持跨数据中心的数据复制
支持镜像集群
```

缺点

```
由于是批量发送，数据达不到真正的实时
对于mqtt协议不支持
不支持物联网传感数据直接接入
只能支持统一分区内消息有序，无法实现全局消息有序
监控不完善，需要安装插件
需要配合zookeeper进行元数据管理
会丢失数据，并且不支持事务
可能会重复消费数据，消息会乱序，可以保证一个固定的partition内部的消息是有序的，但是一个topic有多个partition的话，就不能保证有序了，需要zookeeper的支持，topic一般需要人工创建，部署和维护一般都比mq高
```

#### <a name="3">Kafka的使用场景


1、消息队列功能：在系统或应用程序之间构建可靠的用于传输实时数据的管道

2、数据处理功能：在系统或应用程序之间构建可靠的用于传输实时数据的管道，

```
日志收集：一个公司可以用kafka收集各种服务的log，同kafka以统一接口服务的方式开发给各种consumer

消息系统：解耦生产者和消费者、缓存消息等

流式处理：Spark streaming和Flink

运营指标：记录运营监控数据，包括收集各种分布式应用的数据，生产各种操作的集中反馈，比如报警和报告

用户活动跟踪：记录web用户或者app用户的各种活动，比如浏览网页、搜索、点击等活动，这些活动信息被各个服务器发布到kafka的topic中，然后消费者通过订阅这些topic来做实时的监控分析，亦可保存到数据库
```

#### <a name="4">Kafka架构


<img src="https://yingziimage.oss-cn-beijing.aliyuncs.com/img/image-20220407200703144.png" alt="image-20220407200703144" style="zoom:67%;" />

```
Producer：消息生产者，向kafka broker发消息的客户端

Consumer：消息消费者，向kafka broker取消息的客户端

Consumer Group(CG)：消费者组，由多个consumer组成。消费者组内每个消费者负责消费不同分区的数据，一个分区只能由一个消费者消费；消费者之间互不影响。所有的消费者都属于某个消费者组，即消费者组是逻辑上的一个订阅者

Broker：一台kafka服务器就是一个broker，一个集群由多个broker组成，一个broker可以容纳多个topic

Topic：可以理解为一个队列，生产者和消费者面向的都是topic

Partition：为了实现扩展性，一个非常大的topic可以分布到多个broker上，一个topic开源分为多个partition，每个partition是一个有序的队列

Replica：副本，为保证集群中的某个节点发送故障时，该节点上的partition数据不丢失，且kafka仍然能够继续工作，kafka提供了副本机制，一个topic的每个分区都有若干个副本，一个leader和若干个follower

leader：每个分区多个副本的“主，生产者发送数据的对象，以及消费者消费数据的对象都是leader

follower：每个分区多个副本的“从”，实时从leader中同步数据，保持和leader数据的同步。leader发生故障时，某个follower会成为新的leader
```

### <a name="5">Kafka的生产者区域


#### <a name="6">分区策略


1）分区的原因

```
1、方便在集群中扩展，每个Partition可以通过调整以适应它所在的机器，而一个topic又可以由多个Partition组成，因此整个集群就可以适应任意大小的数据了
2、可以提高并发，因为可以以Partition为单位读写
```

2）分区的原则

![image-20220408122404946](https://yingziimage.oss-cn-beijing.aliyuncs.com/img/image-20220408122404946.png)

```
我们需要将Producer发送的数据封装成一个ProducerRecord对象
	1、指明partition的情况下，将其作为partition值
	2、没有指明partiton值但有key的情况下，将key的hash值与topic的partition数进行取余得到partition值
	3、没有partition、key值情况下。第一次调用时随机生成一个整数，将这个值与topic可用的partition总数取余得到partition值。round-robin算法
```

#### <a name="7">数据可靠性保证


<img src="https://yingziimage.oss-cn-beijing.aliyuncs.com/img/image-20220408122857362.png" alt="image-20220408122857362" style="zoom: 67%;" />

为保证producer发送数据，能可靠的发送到指定的topic，topic的每个partition收到producer发送的数据后，都需要向producer发送ack，如果producer收到ack，就会进行下一轮的发送，否则重新发送数据

##### <a name="8">1）副本数据同步策略


| 方案                        | 优点                                               | 缺点                                                |
| --------------------------- | -------------------------------------------------- | --------------------------------------------------- |
| 半数以上完成同步，就发送ack | 延迟低                                             | 选举新的leader时，容忍n台节点的故障，需要2n+1个副本 |
| 全部完成同步，才发送ack     | 选举新的leader时，容忍n台节点的故障，需要n+1个副本 | 延迟高                                              |

```
Kafka选择了第二种方案，原因如下：
	1.同样为了容忍n台节点的故障，第一种方案需要2n+1个副本，第二种方案只需要n+1个副本，而Kafka的每个分区都有大量的数据，第一种方案会造成大量数据的冗余
	2.虽然第二种方案的网络延迟会比较高，但网络延迟对Kafka的影响较大
```

##### <a name="9">2）ISR机制


ISR：副本同步队列

````
ISR是Leader维护了一个动态副本同步队列，是和leader保持同步的follower集合，ISR中包括Leader和Follower集合，ISR中包括Leader和Follower。当ISR中的follower完成数据的同步之后，leader就会给producer发送ack。如果follower长时间未向leader同步数据，则该follower将被提出ISR，加入到OSR（Out-Sync Relipcas），该时间阈值由replica.lag.time.max.ms参数设定，Leader发送故障之后，就会从ISR中选举新的leader

如果OSR集合follwer副本“追上”了Leader，再加到ISR中，当Leader发送故障时，只有在ISR集合中的副本才有资格被选举为leader，而在OSR集合中的副本则没有机会
````

##### <a name="10">3）ack应答机制


对于某些不太重要的数据，对数据的可靠性要求不是很高，能够容忍数据的少量丢失，所以没必要等ISR中的follower全部接收成功。

​		所以Kafka为用户提供了三种可靠性级别，用户根据对可靠性和延迟的要求进行权衡，有如下参数选择

```
0：producer不等待broker的ack，这一操作提供了一个最低的延迟，broker一接收到还没有写入磁盘就已经返回，当broker故障时有可能丢失数据
1：producer等待broker的ack，partition的leader落盘成功后返回ack，如果再follower同步成功之前leader故障，将丢失数据
-1：producer等待broker的ack，partition的leadr和follower全部落盘成功后才返回ack，但是如果再follower同步完成后，broker发送ack之前，leader发送故障，那么会造成数据重复
```

##### <a name="11">4）故障处理细节


<img src="https://yingziimage.oss-cn-beijing.aliyuncs.com/img/image-20220408123924723.png" alt="image-20220408123924723" style="zoom: 67%;" />

- LEO：每个副本最大的offset
- HW：消费者能见到最大的offset，ISR队列中最小的LEO

```
(1)follower故障
	follower发送故障后会被临时踢出ISR，待该follower恢复后，follower会读取本地磁盘记录的上次的HW,并将log文件高于HW的部分截取掉，从HW开始向leader进行同步。等该follower的LEO大于等于Partition的HW，即follower追上leader之后，就可以重新加入ISR了
(2)leader故障
	leader发送故障之后，会从ISR中选出一个新的leader。之后，为保证多个副本之间的数据一致性，其余的follower会先将各自的log文件高于HW的部分截掉，然后从新的leader同步数据（注：只能保证副本之间的数据一致性，不能保证数据不丢失或不重复）
```

#### <a name="12">Exactly Onec语义


```
ACK = -1,保证Producer->Server之间不丢失数据，即At Least Once语义
ACK = 0,保证生产者每条消息只会被发送一次，即At Most Once语义
幂等性：Producer不论向Server发送多少次重复数据，Server端都只会持久化一条
At Least Once + 幂等性 = Exactly Once
```

#### <a name="13">生产者发送的一条 message 中包含哪些信息


消息可由可变长度的报头不透明密钥字节数组、不透明值字节数组组成

RecordBatch是Kafka数据的存储单元，一个RecordBatch中包含多个Record（消息），每条消息包含多个Header（K-V形式）

#### <a name="14">生产者向Kafka发送消息的执行流程


<img src="https://yingziimage.oss-cn-beijing.aliyuncs.com/img/image-20220409112717289.png" alt="image-20220409112717289" style="zoom:67%;" />

（1）生产者要往Kafka发送消息时，需要创建ProducerRecoder对象

```java
ProducerRecord<String,String> record 
      = new ProducerRecoder<>("CostomerCountry","Precision Products","France");
      try{
      producer.send(record);
      }catch(Exception e){
        e.printStackTrace();
      }
```

（2）ProducerRecoder对象会包含目标topic，分区内容，以及指定的key和value，在发送ProducerRecoder时，生产者会先把键和值对象序列化成字节数组，然后在网络上传输

（3）生产者在将消息发送到某个Topic，需要经过拦截器，序列化器和分区器

（4）如果消息ProducerRecord么有指定partition字段，那么就需要依赖分区器，根据key这个字段来计算partition的值。**分区器的作用就是为消息分配分区**

1.若没有指定分区，且消息的key不为空，则使用hash算法来计算分区分配

2.若没有指定分区，且消息的key也是空，则用轮询的方式选择一个分区

（5）分区选择好之后，会将消息添加到一个记录批次中，这个批次的所有消息都会被发送到相同的Topic和partition上。然后会有一个独立的线程负责把这些记录批次发送到相应的broker中

（6）brker接收到Msg后，会作出一个响应。如果成功写入Kafka中，就返回一个RecordMetaData对象，它包含Topic和Partition信息，以及记录在分区的offset

（7）若写入失败，就返回一个错误异常，生产者在收到错误之后尝试重新发送消息，几次之后如果还失败，就返回错误信息

#### <a name="15">kafka文件存储机制


<img src="https://yingziimage.oss-cn-beijing.aliyuncs.com/img/image-20220409112919953.png" alt="image-20220409112919953" style="zoom:67%;" />

在Kafka中，一个Topic会被分割成多个Partition，而Partition由多个更小的Segment的元素组成。

Kafka会根据log.segment.bytes的配置来决定单个Segment文件(log)的大小，当写入数据达到这个大小时就会创建新的Segment.

<img src="https://yingziimage.oss-cn-beijing.aliyuncs.com/img/image-20220408203528454.png" alt="image-20220408203528454" style="zoom:50%;" />

<img src="https://yingziimage.oss-cn-beijing.aliyuncs.com/img/image-20220408203543538.png" alt="image-20220408203543538" style="zoom:67%;" />

<img src="https://yingziimage.oss-cn-beijing.aliyuncs.com/img/image-20220408203555748.png" alt="image-20220408203555748" style="zoom:67%;" />

log、index、timeindex中存储的都是二进制的数据。具体来说就是：log中存储BatchRecords消息内容，而index和timeindex分别是一些索引信息







### <a name="16">Kafka的消费者区域


#### <a name="17">消费方式


consumer采用push（推）、pull（拉）模式从broker中读取数据

```
push：很难适应消费速率不同的消费者，消息发送速率是由broker决定的。它的目标是尽可能以最快速度传递消息，但是这样很容易造成comsuer来不及处理消息，典型的表现是拒绝服务以及网络拥塞

pull：根据consumer的消费能力以适当的速率消费消息。但如果kafka没有数据，消费者可能会陷入循环中，一直返回空数据。针对这一点，Kafka的消费者在消费数据时会传入一个时长参数timeout，如果当前没有数据可供消费，consumer会等待一段时间之后再返回，这段时长即为timeout
```

#### <a name="18">分区分配策略


一个topic有多个partition，一个consumer group中有多个consumer，那么怎么确定哪个partition由哪个consumer来消费

Kafka有两种分配策略，一种是RoundRobin，Range

<img src="https://yingziimage.oss-cn-beijing.aliyuncs.com/img/image-20220408124933265.png" alt="image-20220408124933265" style="zoom:33%;" />

<img src="https://yingziimage.oss-cn-beijing.aliyuncs.com/img/image-20220408124946492.png" alt="image-20220408124946492" style="zoom:33%;" />

#### <a name="19">kafka的消费者组跟分区之间的关系


```
1、afka 中，通过消费者组管理消费者，假设一个主题中包含 4 个分区，在一个消费者组中只要一个消费者。那消费者将收到全部 4 个分区的消息
2、如果存在两个消费者，那么四个分区将根据分区分配策略分配个两个消费者
3、如果存在四个消费者，将平均分配，每个消费者消费一个分区
4、如果存在5个消费者，就会出现消费者数量多于分区数量，那么多余的消费者将会被闲置
```

#### <a name="20">offset的维护


由于在consumer在消费过程中可能会出现断电宕机等故障，consumer恢复后，需要从故障前的位置继续消费，所以consumer需要实时记录自己消费到哪个offset，以便故障恢复后继续消费。

<img src="https://yingziimage.oss-cn-beijing.aliyuncs.com/img/image-20220408125031731.png" alt="image-20220408125031731" style="zoom: 67%;" />

#### <a name="21">如何实现 kafka 消费者每次只消费指定数量的消息


```
写一个队列，把 consumer 作为队列类的一个属性，然后增加一个消费计数的计数器，当到达指定数量时，关闭 consumer
```

#### <a name="22">kafka如何实现多线程的消费


```
kafka 允许同组的多个 partition 被一个 consumer 消费
不允许一个 partition 被同组的多个 consumer 消费

实现多线程步骤如下：
1、生产者随机分区提交数据(自定义随机分区)。
2、消费者修改单线程模式为多线程，在消费方面得注意，得遍历所有分区，否则还是只消费了一个区
```

#### <a name="23">kafka消费支持几种消费模式


```
at most once模式：最多一次，保证每一条消息commit成功之后，再进行消费处理。消息可能会丢失，但不会重复

at least once模式：至少一次，保证每一条消息处理成功之后，再进行commit。消息不会丢失，但可能会重复

exactly once模式：精确传递一次，将offset作为唯一id与消息同时处理，并且保证处理的原子性。消息只会处理一次，不丢失也不会重复。但这种方式很难做到

kafka默认的模式是 at least once，但这种模式可能会产生重复消费的问题，所以在业务逻辑必须做幂等设计。
```



### <a name="24">综合


### <a name="25">Kafka高效读写数据


1）顺序读写磁盘

```
Kafka 的 producer 生产数据，要写入到 log 文件中，写的过程是一直追加到文件末端， 为顺序写。官网有数据表明，同样的磁盘，顺序写能到 600M/s，而随机写只有 100K/s。这 与磁盘的机械机构有关，顺序写之所以快，是因为其省去了大量磁头寻址的时间
```

2）零复制技术

<img src="https://yingziimage.oss-cn-beijing.aliyuncs.com/img/image-20220408125300963.png" alt="image-20220408125300963" style="zoom: 67%;" />

#### <a name="26">Zookeeper在Kafka中的作用


Kafka集群中有一个broker会被选举为**Controller**，负责管理集群broker的上下线，所有topic的分区副本分配和leader选举等工作。Controller的管理工作都是依赖于Zookeeper的

以下为partition的leader选举过程：

<img src="https://yingziimage.oss-cn-beijing.aliyuncs.com/img/image-20220408163718275.png" alt="image-20220408163718275" style="zoom:67%;" />

#### <a name="27">Kafka事务


Kafka 从 0.11 版本开始引入了事务支持。事务可以保证 Kafka 在 Exactly Once 语义的基 础上，生产和消费可以跨分区和会话，要么全部成功，要么全部失败

**1、Producer事务**

```
	为了实现跨分区跨会话的事务，需要引入一个全局唯一的Transaction ID,并将Producer获得的PID和Transaction ID绑定。这样当Producer重启后就可以通过正在进行的Transaction ID获得原来的PID
	为了管理Transaction,Kafka引入了一个新的组件Transaction Coordinator。Producer 就是通过和 Transaction Coordinator 交互获得 Transaction ID 对应的任务状态。TransactionCoordinator 还负责将事务所有写入 Kafka 的一个内部 Topic，这样即使整个服务重启，由于事务状态得到保存，进行中的事务状态可以得到恢复，从而继续进行
```

**2.Consumer事务**

```
上述事务机制主要是从 Producer 方面考虑，对于 Consumer 而言，事务的保证就会相对较弱，尤其是无法保证 Commit 的信息被精确消费。这是由于 Consumer 可以通过 offset 访问任意信息，而且不同的 Segment File 生命周期不同，同一事务的消息可能会出现重启后被删除的情况
```

#### <a name="28">kafka如何实现消息是有序的


- 生产者：通过分区的leader副本负责数据以先进先出的顺序写入，来保证消息顺序性
- 消费者：同一个分区内的消息只能被一个group里的一个消费者消费，保证分区内消费有序

kafka每个partition中的消息在写入时都是有序的，消费时，每个partition只能被一个消费者组中的一个消费者消费，保证了消费时也是有序的。

整个kafka不保证有序，如果为了保证kafka全局有序，那么设置一个生产者，一个分区，一个消费者

#### <a name="29">kafka的分区算法


1、轮询策略

````
Round-robin策略，顺序分配。比如一个topic下有3个分区，那么第一条消息被发送到分区0，第二条被发送到分区1，第三条被发送到分区2，依次类推。

轮询策略是kafka java生产者API默认提供的分区策略。轮询策略有非常优秀的负载均衡表现，它总是能保证消息最大限度地被平均分配到所有分区上，故默认情况下它是最合理的分区策略，也是平时最常用的分区策略之一
````

2、随机策略

```
Randomness策略，随意地将消息放置在任意一个分区上
```

3、按key分配策略

```
kafka允许为每条消息定义消息键，简称为key。一旦消息被定义了key，那么你就可以保证同一个key的所有消息都进入到相同的分区里面，由于每个分区下的消息处理都是有顺序的
```

#### <a name="30">kafka的默认消息保留策略


broker默认的消息保留策略分为两种：

```
1、日志片段通过log.segment.bytes配置（默认是1GB）
2、日志片段通过log.segment.ms配置（默认是7天）
```

#### <a name="31">kafka如何实现单个集群间的消息复制


```
kafka消息负责机制只能在单个集群中进行复制，不能在多个集群中进行

kafka提供了一个叫做MirrorMaker的核心组件，该组件包含一个生产者和一个消费者，两者之间通过一个队列进行相连，当消费者从一个集群读取消息，生产者把消息发送到另一个集群
```

#### <a name="32">LEO、HW、LSO、LW分别代表什么


```
LEO：LogEndOffset的简称，代表当前日志文件中下一条

HW：水位或水印一词，也可称为高水位，通常被用在流式处理领域（flink、spark），以表征元素或事件在基于时间层面上的进展。在kafka中，水位的概念与时间无关，而是与位置信息相关。严格来说，它表示的就是位置信息，即位移（offset）。取partition对应的ISR中最小的LEO作为HW，consumer最多只能被消费到HW所在的上一条信息

LSO：LastStableOffset的简称，对未完成的事务而言，LSO的值等于事务中第一条消息的位置（firstUnstableOffset），对已完成的事务而言，它的值同HW相同

LW：Low Watermark低水位，代表AR集合中最小的logStartOffset值
```

#### <a name="33">如何保证每个应用程序都可以获取到 Kafka 主题中的所有消息，而不是部分消息


```
为每个应用程序创建一个消费者组，然后往组中添加消费者来伸缩读取能力和处理能力，每个群组消费主题中的消息时，互不干扰
```

#### <a name="34">Kafka的选举机制


在整个系统中，涉及到多处选举，以下从三方面来说明

**1、控制器（Broker）选举**

```
在一个kafka集群中，有多个broker节点，但是它们之间需要选举出一个leader，其他broker充当follower角色。集群中第一个启动的broker会通过在zookeeper中创建临时节点/controller来让自己成为控制器，其他broker启动时也会在zookeeper中创建临时节点，但是发现节点已经存在，所以它们会收到一个异常，意识到控制器已经存在，那么就会在zookeeper中创建watch对象，便于它们收到控制器变更的通知

如果控制器由于网络原因与zookeeeper断开连接或者异常退出，那么其他broker通过watch收到控制器变更的通知，就会去尝试创建临时节点/controller，如果有一个broker创建成功，那么其他broker就会收到创建异常通知，也就意味着集群中已经有了控制器，其他broker只需创建watch对象即可

如果集群中有一个broker发送异常退出，那么控制器就会检查这个broker是否有分区的副本leader，如果有那么这个分区就需要一个新的leader，此时控制器就会去遍历其他副本，决定哪一个成为新的leader，同时更新分区的ISR集合。

如果有个broker加入集群中，那么控制器就会通过Broker ID去判断新加入的broker中是否含有现有分区的副本，如果有，就会从分区副本中去同步数据。

集群中每选举一次控制器，就会通过zookeeper创建一个controller epoch，每一个选举都会创建一个更大，包含最新信息的epoch，如果有broker收到比这个epoch旧的数据，就会忽略它们，kafka也通过这个epoch来防止集群产生脑裂
```

2、分区副本选举机制

```
在kafka的集群中，会存在着多个主题topic，在每一个topic中，又被划分为多个partition，为了防止数据不丢失，每一个partition又有多个副本，在整个集群中，总共有三种副本角色

leader副本：每个分区都有一个leader副本，为了保证数据一致性，所有的生产者与消费者的请求都会经过该副本来处理
follower副本：除了leader副本之外的其他所有副本都是follower副本，follower副本不处理来自客户端的任何请求，只负责从leader副本同步数据，保证与首领保持一致。
优先副本：创建分区时指定的优先leader，如果不指定，则为分区的第一个副本

follower需要从leader中同步数据，但是由于网络或其他原因，导致数据阻塞，出现不一致的情况，为了避免这种情况，follower会想leader发送请求消息，这些请求信息中包含了follower需要数据的偏移量offset，而且这些offset是有序的

如果有follower向leader发送了请求1，接着发送请求2，请求3，那么再发送请求4，这时就意味着follower已经同步了前三条数据，否则不会发送请求4。leader通过跟踪每一个follower的offset来判断它们的复制进度

默认的，如果follower与leader之间超过10s内没有发送请求，或者说没有收到请求数据，此时该follower就会被认为“不同步副本”。而持续请求的副本就是“同步副本”，当leader发生故障时，只有“同步副本”才可以被选举为leader。其中的请求超时时间可以通过参数replica.lag.time.max.ms参数来配置

我们希望每个分区的leader可以分布到不同的broker中，尽可能达到负载均衡，所以会有一个优先leader，如果我们设置参数auto.leader.rebalance.enable为true，那么它会检查优先leader是否是真正的leader，如果不是，则会触发选举，让优先leader成为leader
```

3、消费者选主

````
在kafka的消费端，会有一个消费者协调器以及消费组，组协调器GroupCoordinator需要为消费组内的消费者选举出一个消费组的leaer，那么如何选举的呢？
如果消费组内没有leader，那么第一个加入消费组的消费者即为消费组的leader，如果某一个时刻leader消费者由于某些原因退出了消费组，那么就会重新选举leader
```
private val members = new mutable.HashMap[String,MemberMetadata]
leaderId = members.keys.headOption
```
上面代码是kafka源码中部分代码，member是一个hashmap的数据结构，key为消费者的member_id，value是元数据信息，那么它会将leaderId选举为Hashmap中的第一个键值对，它和随机基本没啥区别。
````

#### <a name="35">kafka如何清理过期数据


kafka将数据持久划到了硬盘上，允许你配置一定的策略对数据清理，清理的策略有两个，删除和压缩

1、删除

```
log.cleanup.policy=delete 启用删除策略
直接删除，删除后的消息不可恢复。可配置以下两个策略
#清理超过指定时间清理：  
log.retention.hours=16
#超过指定大小后，删除旧的消息：
log.retention.bytes=1073741824
为了避免在删除时阻塞读操作，采用了copy-on-write形式的实现，删除操作进行时，读取操作的二分查找功能实际是在一个静态的快照副本上进行的，这类似于Java的CopyOnWriteArrayList
```

2、压缩

```
将数据压缩，只保留每个key最后一个版本的数据
在broker的配置中设置log.cleaner.enable = true（默认false）
在topic的配置中设置log.cleanup.policy=compact启用压缩策略
```
