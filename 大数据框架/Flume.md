&emsp;<a href="#0">Flume</a>  
&emsp;&emsp;<a href="#1">1、什么是Flume</a>  
&emsp;&emsp;<a href="#2">2、Flume优缺点</a>  
&emsp;&emsp;<a href="#3">3、Flume架构</a>  
## <a name="0">Flume


### <a name="1">1、什么是Flume


​		Flume是Cloundera提供的一个高可用的，高可靠的，分布式的海量日志采集、聚合和传输的系统。2009年被捐赠给了apache软件基金会，为hadoop相关组件之一。尤其近几年随着flume的不断被完善以及升级版本的逐一推出，特别是flume-ng，同时flume内部的各种组件不断丰富，用户在开发的过程中使用的便利性得到很大的改善，现已成为apache top项目之一。

​		Flume可以收集例如日志，事件等数据资源，并将这些数量庞大的数据从各项数据资源中集中起来存储的工具/服务。flume具有高可用、分布式、配置工具，其设计的原理是基于数据量（流式架构，灵活简单），如日志数据从各种网站服务器上汇集起来存储到HDFS，HBase上。结构如下：

![](https://yingziimage.oss-cn-beijing.aliyuncs.com/img/202302151634642.png)

Flume最主要的作用是实时读取服务器本地磁盘的数据，将数据写入HDFS或Kafka等

### <a name="2">2、Flume优缺点


```
优点：
1、可将应用产生的数据存储到任何集中存储器中，如HDFS,HBase
2、Flume的管道是基于事务，保证了数据在传送和接收时的一致性
3、Flume是可靠的，容错性高的，可升级的，易管理的，并且可定制的
4、提供上下文路由特征
5、实时性，实时的分析数据并将数据保存在数据库或其他系统中
6、当收集数据的速度超过写入数据的时候(收集信息遇到峰值),这时候收集的信息非常大，甚至超过了系统的写入数据能力，这时候，Flume会在数据生产者和数据收容器间做出调整，起到缓冲的作用。

缺点：
Flume配置很繁琐，source,channel,sink的关系在配置文件里面交织在一起，不便于管理
```

### <a name="3">3、Flume架构


![](https://yingziimage.oss-cn-beijing.aliyuncs.com/img/202302151634599.png)

Agent

```
Agent是一个JVM进程，它以事件的形式将数据从源头送至目的
主要由3个部分组成：Source，Channel，Sink
```

Source

```
负责数据的产生或搜集，一般对接一些PRC的程序或者是其他的flume节点的sink,从数据发生器接收数据，并将接收的数据以Flume的event格式传递给一个或者多个通道Channel,Flume提供多种数据接收的方式。
```

| 类型               | 描述                                                 |
| ------------------ | ---------------------------------------------------- |
| Avro               | 监听Avro端口并接收Avro Client的流数据                |
| Thrift             | 监听Thrift端口并接收Thrift Client的流数据            |
| Exec               | 基于Unix的command在标准输出上产生数据                |
| JMS                | 从JMS（Java消息服务）采集数据                        |
| Spooling Directory | 监听指定目录                                         |
| Twitter 1%         | 通过API持续下载Twitter数据                           |
| NetCat             | 监听端口（数据是换行符分隔的文本）                   |
| Sequence Generator | 序列产生器，连续不断产生event，用于测试              |
| Syslog             | 采集syslog日志消息，支持单（多）端口TCP和UDP日志采集 |
| HTTP               | 接收HTTP POST和GET数据                               |
| Stress             | 用于source压力测试                                   |
| Legacy             | 向下兼容，接收低版本Flume数据                        |
| Custom             | 自定义source的接口                                   |
| Scribe             | 从facebook Scribe采集数据                            |

Channel

```
	Channel是一种短暂的存储容器，负责数据的存储持久化，可以持久化到jdbc、file、memory，将从source处接收到的event格式的数据缓存起来，直到他们被sinks消费掉。可以把channel看成是一个队列，Flume比较看重数据的传输，因此几乎没有数据的解析预处理。仅仅是数据的产生，封装成event然后传输。数据只有存储在下一个存储位置，数据才会从当前的channel中删除。这个过程是通过事务来控制的，这样就保证了数据的可靠性。
	Channel是线程安全的，可以同时处理几个Source的写入及Sink的读取操作
```

| 类型               | 描述                                             |
| ------------------ | ------------------------------------------------ |
| Memory             | 内存                                             |
| JDBC               | 持久化存储中，当前Flume Channel内置支持Derby     |
| Kafka              | kafka集群                                        |
| File               | 磁盘文件中                                       |
| Spillable Memory   | 内存和磁盘上，当内存队列满了，会持久化到磁盘文件 |
| Pseudo Transaction | 测试用途                                         |
| Custom             | 自定义Channel实现                                |

注意：

- Memory：内存中的队列，在不需要关心数据丢失的情景下适用，一旦程序死亡、机器重启都会导致数据丢失
- File：将所有事件写到磁盘。一般不会丢失数据

Sink

```
Sink不断轮询Channel中的事件且批量地移除它们，并将这些事件批量写入到存储或索引系统、或者发送到另一个Flume Agent
```

| 类型           | 描述                                                |
| -------------- | --------------------------------------------------- |
| HDFS           | 数据写入HDFS                                        |
| HIVE           | 数据写入HIVE                                        |
| Logger         | 数据写入日志文件                                    |
| Avro           | 数据被转换成Avro Event，然后发送到配置的RPC端口上   |
| Thrift         | 数据被转换成Thrift Evenr，然后发送到配置的RPC端口上 |
| IRC            | 数据再IRC上进行回放                                 |
| File Roll      | 数据在IRC上进行回收                                 |
| File Roll      | 存储数据到本地文件系统                              |
| Null           | 丢弃所有数据                                        |
| HBase          | 写入HBase数据库                                     |
| Morphline Solr | 数据发送到Solr搜索服务器（集群）                    |
| ElasticSearch  | 数据发送到Elastic Search搜索服务器（集群）          |
| Kite Dataset   | 数据写到Kite Dataset，试验性质的                    |
| Kafka          | 数据写入Kafka Topic                                 |
| Custom         | 自定义Sink实现                                      |

Event

```
传输单元，Flume数据传输的基本单元，以Event的形式将数据从源头送至目的地。Event由Header和Body两部分组成，Header用来存放该event的一些属性，为K-V结构；Body用来存放该条数据，形式为字节数组
```

![](https://yingziimage.oss-cn-beijing.aliyuncs.com/img/202302151634961.png)
