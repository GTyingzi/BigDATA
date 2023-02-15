&emsp;<a href="#0">HBase</a>  
&emsp;&emsp;<a href="#1">HBase介绍</a>  
&emsp;&emsp;<a href="#2">HBase优缺点</a>  
&emsp;&emsp;<a href="#3">HBase数据结构</a>  
&emsp;&emsp;<a href="#4">HBase原理</a>  
&emsp;&emsp;<a href="#5">HBase架构（重点）</a>  
&emsp;&emsp;<a href="#6">HBase核心原理</a>  
&emsp;&emsp;<a href="#7">HBase写流程(重点)</a>  
&emsp;&emsp;<a href="#8">HBase读流程(重点)</a>  
&emsp;&emsp;<a href="#9">HBase的读写缓存</a>  
&emsp;&emsp;<a href="#10">HBase的数据删除</a>  
&emsp;&emsp;<a href="#11">HBase的RegionServer宕机以后怎么恢复</a>  
&emsp;&emsp;<a href="#12">HBase HA的实现(重点)</a>  
&emsp;&emsp;<a href="#13">HBase的rowkey设计原则(重点)</a>  
&emsp;&emsp;<a href="#14">HBase的热点问题</a>  
&emsp;&emsp;<a href="#15">HBase的大合并、小合并</a>  
&emsp;&emsp;<a href="#16">HBase数据的compact流程</a>  
&emsp;&emsp;<a href="#17">HBase的LSM结构</a>  
&emsp;&emsp;<a href="#18">HBase的Get和Scan的区别</a>  
&emsp;&emsp;<a href="#19">HBase和关系型（传统数据库）的区别？</a>  
## <a name="0">HBase




### <a name="1">HBase介绍


HBase是一种分布式、可扩展、支持海量数据存储的非关系型分布式数据库，它参考了谷歌的BigTable建模，主要用来存储非结构化和半结构化的松散数据，是Apache软件基金会的Hadoop项目的一部分，运行于HDFS文件系统之上，可以容错地存储海量稀疏的数据。

HBase的目标是处理非常庞大的表，通过水平扩展的方式，利用廉价计算机集群处理超过10亿行数据和数百万列元素组成的数据表。

### <a name="2">HBase优缺点


优点

```
海量存储：HBase适合存储PB级别的海量数据，可采用廉价PC存储，能在几十到百毫秒返回数据。

列式存储：也就是列族存储(ColumnFamily)存储，列族下面可以有非常多的列。

极易扩展：基于上层处理能力（RegionServer）的扩展；基于存储的扩展（HDFS）
		通过横向添加RegionServer的机器，进行水平扩展，提升HBase上层的处理能力
		
高并发（多核）：在并发的情况下下，HBase的单个IO延迟下降并不多，能获得高并发、低延迟的服务。

稀疏：主要针对HBase列的灵活性，在列族中，你可以指定任意多的列，在列数据为空的情况下，是不会占用存储空间的。
```

缺点

```
原生不支持SQL：Hbase是一个非关系型数据库但不支持SQL语句，不过可以通过Phoenix解决，专门为HBase设计的SQL层

原生不支持二级索引：单一RowKey固有的局限性决定了它不可能有效地支持多条查询，只支持按照Rowkey来查询，因此正常情况下对非Rowkey列做查询比较慢。所以，我们一般会选择一个HBase二级索引解决方案，目前比较成熟的方案是Phoenix，此外还可以选择Elasticsearch/Solr等搜索引擎自己设计实现

暂时不能支持Master server的故障切换，当Master死亡后，整个存储系统会挂掉

数据分析能力弱：在HBase之上架设Phoenix或Spark等组件，增强Hbase数据分析处理的能力。
```

### <a name="3">HBase数据结构


```
1、Name Space
类似于关系型数据库的database。HBase自带两个命名空间。hbase：存放HBase内置表；default：用户默认使用

2、Table
类似于关系型数据库的表。定义表时只需要声明列族即可，超时时间(TTL)、压缩算法(COMPRESSION)等都在列族中定义
往HBase写入数据时，字段可以动态、按需指定

3、Row
由一个RowKey和多个Column组成

4、RowKey
由用户指定唯一的字符串定义。数据按照RowKey的字典顺序存储，查询数据时根据RowKey进行检索

5、Column Family
列族是列的集合，表的相关属性大部分定义在列族上，同一个表里的不同列族可以由完全不同的属性配置。HBase会把相同列族的列尽可能的放在同一台机器上

6、Column Qualifier
列随意定义，但必须依赖于列族存在，名称前带所属的列族，如info : name，infor : age

7、TimeStamp
用于表示数据的不同版本，时间戳默认由系统指定。读取单元格时，HBase默认读取最后一个版本数据

8、Cell
一个列可以存储多个版本的数据，每个版本称为一个Cell
Cell由{rowkey,column Family : column Qualifier,time Stamp}确定，由字节码存贮

9、Region
Region由一个表的若行组成，在Region中行的排序按照行键(rowkey)字典排序
```

### <a name="4">HBase原理


HBase是大数据NoSQL领域里非常重要的分布式KV数据库，是一个高可靠、高性能、高伸缩的分布式存储系统。

![](https://yingziimage.oss-cn-beijing.aliyuncs.com/img/image-20220313154008598.png)

Master

```
1.用于协调多个Region Server，侦测多个Region Server之间的状态，并平衡Region Server之间的负载。
2.分配Region给RegionServer
3.允许多个Master节点共存，但是需要Zookeeper的帮助，只有一个Master提供服务。
```

Region Server

```
	包括了多个Region,Region Server的作用只是管理表格，以及实现读写操作。Client直接连接Region Server，并通信获取HBase中的数据，对于Region而言，则是真实存放HBase数据的地方，也就说Region是HBase可用性和分布式的基本单位。
	如果当一个表格很大，并由多个CF组成时，那么表的数据将存放在多个Region之间，并且每个Region中会关联多个存储的单元（Store）
```

Zookeeper

```
1、作为HBase Master的HA的解决方案
2、负责Region和Region Server的注册
```

### <a name="5">HBase架构（重点）


<img src="https://yingziimage.oss-cn-beijing.aliyuncs.com/img/image-20220313162947550.png" alt="image-20220313162947550" style="zoom:80%;" />

HBase中的存储包括：HMaster、HRegionSever、HRegion、HLog、Store、MemStore、StoreFile、HFile等

HBase中的每张表都通过键安装一定的范围被分割成多个子表（HRegion），默认一个HRegion超过256M就要被分割成两个，这个过程由HRegionServer管理，而HRegion的分配由HMaster管理

HMaster的作用

```
1、为HRegionServer分配HRegion
2、负责HRegionServer的负载均衡
3、发现失效的HRegionServer并重新分配
4、HDFS上的垃圾文件回收
5、处理Schema更新请求
```

HRegionServer的作用

```
1、维护HMaster分配给它的HRegion,处理HRegion的IO请求
2、负责切分正在运行过程中变得更大的HRegion

通过架构可以得知，Client访问HBase上的数据并不需要HMaster参与，寻址访问ZooKeeper和HRegionServer，数据读写访问HRegionServer，HMaster仅仅维护Table和Region的元数据信息，Table的元数据信息保存在ZooKeeper上，负载很低。
HRegion存储一个子表时，会创建一个HRegion对象，然后对表的每个列簇创建一个Store对象，每个Store都会有一个MemStore和相应个数的StoreFile（对应HFile）

一个HRegionServer会有多个HRegion和一个HLog
```

HRegion

```
Table在行的方向上分割为多个HRegion，HRegion是HBase中分布式存储和负载均衡的最小单元，即不同的HRegion可以分别在不同的HRegionServer上，但是一个HRegion是不会拆分到多个HRegionServer上的。

HRegion按大小分割，每个表一般只有一个HRegion，随着数据不断插入表，HRegion不断增大，当HRegion的某个列簇达到一个阈值（默认256M）时就会分成两个新的HRegion。

HRegion被分配给哪个HRegionServer是完全动态地，所以需要机制来定位HRegion具体在哪个HRegionServer，Hbase使用三层结构来定位HRegion
	1.通过zk里的文件/Hbase/rs得到-ROOT-表位置，-ROOT-表只有一个region
	2.通过-ROOT-表查找.META.表的第一个表中相应的HRegion
	3.通过.META.表找到所要的用户表HRegion的位置
```

Store

```
每一个HRegion由一个或多个Store组成，HBase会把一起访问的数据放在一个Store里，为每个列簇创建一个Store，一个Store由一个MemStore和相应个数的StoreFile组成。
```

MemStore

```
MemStore是放在内存里的，保存修改的数据即keyValues。当MemStore的大小达到一个阈值（默认为64M），MemStore会被Flush到文件，即生成一个快照。目前HBase有一个线程来负责MemStore的Flush操作
```

StoreFile：MemStore内存中的数据写到文件后就是StoreFile，StoreFile底层是以HFile的格式保存

HFile

```
HBase中KeyValue数据的存储格式，是Hadoop的二进制格式文件。HFile文件是不定长的，长度固定只有：Trailer和FileInfo。
Trailer中有指针指向其他数据库的起始点，FileInfo记录了文件的一些meta信息。
Data Block是HBase IO的基本单元，为了提高效率，HRegionServer中有基于LRU的Block Cache机制，每个Data块的大小可以在创建一个Table的时候通过参数指定（64KB），大号的Block有利于顺序Scan，小号的Block利于随机查询。每个Data块除了开头的Magic以外就是一个个KeyValue对拼接而成，Magic内容就是一些随机数字，目的是防止数据损坏，结构如下
```



<img src="https://yingziimage.oss-cn-beijing.aliyuncs.com/img/image-20220314103313368.png" alt="image-20220314103313368" style="zoom:67%;" />

HLog：用来做灾难恢复使用，HLog记录数据的所有变更，一旦region server宕机，就可以从log中进行恢复

### <a name="6">HBase核心原理


1、存储引擎

HBase是Google的BigTable的开源实现，底层引擎是基于LSM-Tree数据结构设计。写入数据时会先写WAL日志，再将数据写到缓存MemStore中，等写缓存达到一定规模或满足其他触发条件会flush刷写到磁盘，这样将磁盘随机写变成了顺序写，提高了写性能。每一次刷写磁盘都会生成新的HFile文件

![](https://yingziimage.oss-cn-beijing.aliyuncs.com/img/image-20220313160953344.png)

随着时间推移，写入的HFile会越来越多，查询数据时就会因为要进行多次io导致性能降低，为了提升读性能，HBase会定期执行compaction操作以合并HFile。此外，HBase在读路径上也有诸多设计，其中一个重要的点设计了BlockCache读缓存。这样以后，读取数据时会依次从BlockCache、MemStore以及HFile中seek数据，再加上一些其他设计比如布隆过滤器、索引等，保证了HBase的高性能。

2、数据模型

关于HBase的数据模型，和关系型数据类似，包括命名空间（namespace）、表、行、列、列族、列限定符、单元格（cell）、时间戳等。HBase在实际存储数据的时候是以有序KV的形式组织的

<img src="https://yingziimage.oss-cn-beijing.aliyuncs.com/img/image-20220313161552389.png" alt="image-20220313161552389" style="zoom:67%;" />

这里重点从KV这个角度切入，Value是实际写入的数据，比较好理解。

```
rowkey：HBase的行键
Column family（列族）与qualifier（列名）共同组成了HBase的列
timestamp：数据写入时的时间戳，主要用于标识HBase数据的版本号
type：代表Put/Delete的操作类型。（注：HBase删除是给数据打上delete marker，在数据合并时才会真正物理删除）
```

3、列族式存储

```
HBase是面向列的列族式存储。前面也提到了，HBase的每一列数据的底层都是以KV形式存储的
```

4、关于索引

```
默认情况下HBase只对rowkey做了单列索引，所以HBase能通过rowkey进行高效的单点查询及小范围扫描。HBase索引是比较单一的，通过非rowkey列查询性能比较低，除非对非Rowkey列做二级索引，否则不建议根据非rowkey列做查询
HBase的二级索引一般是基于HBase协处理器实现，目前成熟的方案是Phoenix，扮演了HBase的SQL层，增强了HBase的即席查询的能力
```

5、应用场景

```
HBase经常应用在订单/消息存储、用户画像、搜索推荐、社交Feed流、安全风控、以及物联网时序数据等诸多场景
```



### <a name="7">HBase写流程(重点)


<img src="https://yingziimage.oss-cn-beijing.aliyuncs.com/img/image-20220314103953633.png" alt="image-20220314103953633" style="zoom:80%;" />

```
1、Client先访问zookeeper，获取hbase:meta表位于哪个Region Server
2、访问对应的RegionServer，获取hbase:meta表，根据读请求的namespace:table/rowkey，查询出目标数据位于哪个RegionServer中的哪个Region中。并将该table的region信息以及meta表的位置信息缓存在客户端的meta cache，方便下次访问。
3、与目标RegionServer进行通讯
4、将数据顺序写入（追写）到WAL
5、将数据写入对应的MemStore，数据会在MemStore进行排序
6、RegionServer向Client发送ack
7、等达到MemStore的刷写时机后，将数据刷写到HFile
```

### <a name="8">HBase读流程(重点)


<img src="https://yingziimage.oss-cn-beijing.aliyuncs.com/img/image-20220314104734213.png" alt="image-20220314104734213" style="zoom: 67%;" />

<img src="https://yingziimage.oss-cn-beijing.aliyuncs.com/img/image-20220314104752623.png" alt="image-20220314104752623" style="zoom: 50%;" />

```
1、Client先访问zookeeper，获取hbase:meta表位于哪个Region Server
2、访问对应的Region Server，获取hbase:meta表，根据读请求的namespace:table/rowkey，查询出目标数据位于哪个Region Server中的哪个Region中。并将table的region信息以及meta表的位置信息缓存在客户端的meta cache，方便下次访问
3、与目标Region Server进行通讯
4、分别在Block Cache（读缓存）、MemStore和Store File（HFile）中查询目标数据，并将查到的所有数据进行合并。此处所有数据是指同一条数据的不同版本（time stamp）或者不同的类型（Put/Delete）
5、将查询到的数据库（Block、HFile数据存储单元，默认大小为64KB）缓存到Block Cache
6、将合并后的最终结果返回给客户端
```

### <a name="9">HBase的读写缓存


HBase上的RegionServer的cache主要分为两个部分：MemStore（写缓存）、BlockCache（读缓存）

**MemStore**

```
当数据写入HBase时，会先写入memstore。RegionServer会先给region提供一个memstore，当memstore中的数据达到系统设置的阈值后，会触发flush将memstore中的数据刷写到HFile

cline的读请求会依次从memstore、blockcache、HFile读取
并同时将读入的数据放入blockcache，BlockCache采用LRU策略，当达到heap*hfile.block.cache.size*0.85后淘汰一批老数据
```

**BlockCache**

```
分两类：JVM的heap内存、heap off内存

第一类策略是LRUBlockCache
HBase默认的BlockCache机制，使用一个ConcurrentHashMap管理BlockKey到Block的映射关系，将BlockKey和对应的Block放到对应的HashMap中，查询缓存就从HashMap中查询BlockKey。
将缓存分为三块：single-access区（25%）、mutil-access区（50%）、in-memory区（25%），核心思想是将Cache分级，避免Cache之间相互影响
single-access：当一个数据块第一次从HDFS读取时，它会具有这种优先级，并且在缓存空间需要被回收（置换）时，优先被考虑。它的优点在于，一般被扫描读取读取的数据块，相较于之后被用到的数据块，更应该被优先清除
mutil-access：一个数据块属于single-access优先级，但是之后被再次访问，则会升级为mutil-access。缓存空间清除（置换）时次要被考虑
in-access：数据常驻内存，用来存放访问频繁、数据量小的数据，比如元数据。


第二类策略是SlabCache、BucketCache
1、SlabCache
内部结构划分两块，80%和20%；缓存数据<=blocksize，则放入80%的区域；1x<缓存数据<2x，则放入20%区域；缓存数据>=2x不缓存
同样使用LRU算法对过期Block进行淘汰，将对应的bufferbyte标记为空闲，后序cache对其上的内存直接覆盖
HBase实际实现中将SlabCahe和LRUBlockCache搭配使用，称为DoubleBlockCache。一次随机读中，一个Block块从HDFS中加载出来之后会在两个Cache中分别存储一份；缓存读首先在LRUBlockCache中查找，如果Cache miss再在SlabCache查找，找到将该Block缓存一份给LRUBlockCache
弊端：SlabCache设计中固定大小内存设置会导致实际内存使用率比较低，而且使用LRUBlockCache缓存Block会因为JVM GC产生大量内存碎片。

2、BucketCache
三种工作模式：heap，offheap，file
BucketCache在初始化的时候申请14个不同大小的Bucket，某一种Bucket空间不足时系统会从其他Bucket空间借用内存。
heap：JVM Heap
offheap：DirectByteBuffer技术实现堆外内存存储管理
file：类似SSD的高速缓存文件存储数据块
HBase将BucketCache和LRUCache搭配使用，称为CombinedBlockCache。系统在LRUBlockCache中主要存储Index Block和Bloom Block，BucketCache存储Data Block。因此一次随机读首先找到Index Block然后在查找对应的Data Block。
弊端：使用堆外内存会存在拷贝内存的问题，一定程度上汇影响读写性能
```

### <a name="10">HBase的数据删除


```
1、Flush删除
Flush只会删除当前memStore中重复的数据（timestamp较小的删除）
StoreFile重复的不会被删除；被标记为DeleteColumn的不会被删除

2、Major Compact删除
当文件数>=3时，会将全部重复的数据进行删除，包括StoreFile；将被标记为DeleteColumn的删除

问：在建表时指定2个版本，put进去相同rowkey的数据，只会保留两个timestamp大的。为什么flush时被标记为删除的数据未删？
答：如果flush后被标记的数据删除了，StoreFile中有相同的rowkey的数据。此时查看rowkey的数据仍然显示，就不正常了。
```

### <a name="11">HBase的RegionServer宕机以后怎么恢复


常见故障

```
1、FullGc引起长时间停顿
2、HBase对JVM堆内存管理不善，未合理使用堆外内存
3、JVM启动参数配置不合理
4、业务写入或吞吐量太大
5、写入读取字段太大
6、HDFS异常
7、机器宕机
	物理节点直接宕机
	虚拟云主机不稳定，包括网络环境等
```

故障恢复

```
Master恢复：
Master主要负责实现集群的负载均衡和读写调度，没有直接参与用户的请求，所以整体负载并不高
热备方式发现Master高可用，zookeeper上进行注册
active master会接管整个系统的元数据管理任务，zk以及meta表中的元数据，相应用户的管理指令，创建、删除、修改，merge region等

Region Server恢复：
RegionServer宕机，HBase会检测到
Master将宕机RegionServer上所有region重新分配到集群中其他正常的RegionServer上
根据HLog进行丢失数据恢复，恢复之后对外提供服务
```

大致流程：

```
1、master通过zk实现对RegionServer的宕机检测，RegionServer会周期性的向zk发送心跳，超过一定时间,zk会认为RegionServer离线，发送消息给master
2、切分为持久化的日志，所有region的数据都混合存储在同一个HLog文件里，为了使这些数据能够安装region进行组织回放，需要将HLog日志进行切分再合并，同一个region的数据合并在一起，方便后续安装region进行数据恢复。
3、master重新分配宕机regionserver上的所有region,regionserver宕机后，所有region处于不可用状态，所有路由到这些region上的请求都会返回异常。异常情况比较短暂，master会将这些region分配到其它regionserver上
4、回放HLog日志补数据
5、恢复完成，对外提供读写服务

流程总结如下：故障检测 -> 数据切分 -> region上线 -> 数据回放
```

### <a name="12">HBase HA的实现(重点)


没有高可用的HBase服务会出现哪些问题：

- 数据库管理人员失误，进行了不可逆的DDL操作
- 离线MR消耗过多的资源，造成线上服务受到影响
- 不可预计的另外一些情况：比如核心交换机故障，机房停电等等情况都会造成HBase服务中断

```
1、不可逆DDL问题
HBase的高可用不支持DDL操作。在master上的DDL操作，不会影响到slave上的数据
2、离线MR影响线上业务问题
高可用的最大好处就是可以进行读写分离，离线MR可以直接跑在slave上，master继续对外提供写服务。HBase的高可用复制是异步进行的
3、意外情况
对于像核心机房故障、断电等意外情况，slave跨机架或者跨机房部署都能解决这种情况

基于以上原因，如果是核心服务，对于可用性要求非常高，可以搭建HBase的高可用来保障服务较高的可用性，在HBase的Master出现异常时，只需简单把流量切换到Slave上，即可完成故障转移，保证服务正常运行
```

<img src="https://yingziimage.oss-cn-beijing.aliyuncs.com/img/image-20220329161838827.png" alt="image-20220329161838827"  />

```
HBase的replication以Column Family为单位，每个Column Family都可以设置是否进行replication.
上图中1个Master对应3个Slave.在开启Replication的情况下，每个RegionServer都会开启一个线程用于读取该RegioServer的HLog，并且发送到各个Slave，zk用于保存当前已经发送的Hlog的位置。Master与Slave之间采用异步通信的方式，保障Master上的性能不会受到Slave的影响。
HBase Replication步骤：
1、HBase Client向Master写入数据
2、对应RegionServer写完HLog后返回Client请求
3、同时replication线程轮询HLog发现有新的数据，发送给Slave
4、Slave处理完数据后返回给Master
5、Master收到Slave的返回信息，在zk中标记已经发送的HLog位置
```

### <a name="13">HBase的rowkey设计原则(重点)


在HBase中，表会被划分为1...n个Region，被托管在RegionServer中。Region两个重要的属性：StartKey与EndKey表示这个Region维护的rowKey范围，当我们要读/写数据时，如果rowKey落在某个start-end key范围内，那么就会定位到目标region并且读/写到相关的数据

**设计原则如下**

1）rowkey长度原则

Rowkey是一个二进制码流，Rowkey的长度被很多开发者建议说设计在10~100个字节，建议是越短越好，不要超过16个字节.

```
原因如下：
1、数据的持久化文件HFile中是按照Key Value存储的，如果Rowkey过长，比如100个字节，1000万列数据。光Rowkey就要占有100*1000万=10亿个字节，这会极大影响HFile的存储效率
2、MemStore将部分数据缓存到内存，如果Rowkey字段过长内存的有效利用率会降低，系统将无法缓存更多的数据，这会降低检索效率
3、目前操作系统都是64位，内存8字节对齐。控制在16个字节，8字节的整数倍利用操作系统的最佳特性
```

2）rowkey散列原则

```
如果rowkey是按时间戳的方式递增，不要将时间放在二进制码的前面，
建议将rowkey的高位作为散列字段，由程序循环生成，低位放时间字段，将会提高数据均衡分布在每个Regionserver实现负载均衡的几率。
如果没有散列字段，首字段直接是时间信息将产生所有新数据都在一个RegionServer上堆积的热点现象，这样在做数据检索的时候负载将会集中在个别RegionServer，降低查询效率
```

3）rowkey唯一原则

```
必须在设计上保证其唯一性。rowkey是按照字典顺序排序存储的，将经常读取的数据存储到一块，将最近可能会被访问的数据放到一块。
```

### <a name="14">HBase的热点问题


1）什么是热点

HBase中的行安装rowkey的字典顺序排序的，这种设计优化了scan操作，可以将相关的行以及会被一起读取的行存储在临近位置，便于scan。

2）热点产生的原因

```
1、HBase中的数据是按照字典排序的，当大量连续的rowkey集中在个别的region，各个region之间的数据分布不均衡
2、创建表时没有提前预分区，创建的表默认只有一个region，大量的数据写入写入当前region
3、创建表已经提前预分区，但是设计的rowkey没有规律可循，设计的rowkey应该由regionNo + messageId组成
```

3）热点解决方案

```
1、HBase创建表时指定分区：HBase预分区
2、合理设计rowkey：参考rowkey设计原则
```

4）HBase常见避免热点问题的方法

```
1、加盐：添加rowkey前缀，决定了在哪一个分区
2、哈希：哈希会使同一行永远用一个前缀加盐。哈希也可以使负载分散到整个集群，但是读却是可以预测的。使用确定的哈希可以让客户端重构完成的rowkey，使用Get操作获取正常的某一行数据
3、反转：反转固定长度或数字格式的rowkey，这样可以使得rowkey中经常改变的部分放在签名，这样可以有效地随机rowkey，但是牺牲了rowkey的有序性
```



### <a name="15">HBase的大合并、小合并


删除一条记录就会在该记录上打上标记DeleteColumn，该记录使用get和scan查询不到，但还是在HFile中。只有进行大合并时才会删除这条记录

```
大合并：region的一个列族所有HFile合并成一个HFile
小合并：多个小的HFile合并成一个大的HFile，将新文件设置为激活状态，删除小文件
```

### <a name="16">HBase数据的compact流程


牺牲磁盘IO来换读性能的基本稳定

```
合并流程：
1、HBase触发Compaction的条件
	-Memstore Flush（每次memstore在flush之后都会判断触发Compaction）
	-后台线程周期性检查
	-手动触发
2、HBase单独有一个线程从Store中选择合适的HFile
3、针对小大合并、split等操作都有对应的线程池进行处理
4、分别读取待合并的HFile文件的数据（K,V），进行归并排序写入./tmp临时文件中
5、将临时文件移到对应的Store的数据目录
6、将Compaction的输入路径和输出路径封装在KV写入到HLog日志，打上Compaction标记，最后强制执行sync
7、将对应的Store数据目录下的Compaction输入文件全部删除

优缺点：
	1、尽量提高数据的本地化率，降低数据读取响应延时，减少网络IO。因为有些文件是在远程节点存储，通过Compaction会尽量本地化
	2、牺牲短时间的性能资源换取后续查询的稳定
	3、在Compaction过程中会有带宽压力和IO压力
	4、在Compaction过程中会对写请求造成阻塞，如HFile较多时，达到默认配置会限制写请求的速度或者短时间的阻塞。
```

### <a name="17">HBase的LSM结构


先了解下三种基本的存储引擎

```
哈希存储引擎：哈希表的持久化实现，支持增删改查，但不支持顺序扫描，对应的存储系统为key-value存储系统，对于key-value的插入及查询，哈希表的复杂度都是O(1)，明显比树的O(n)快

B树存储引擎：B树的持久化实现，支持增删改查，支持顺序扫描，对应的存储系统是关系数据库

LSM树存储引擎：支持增删改查，支持顺序扫描操作。通过批量存储技术规避磁盘随机写入问题
```

<img src="https://yingziimage.oss-cn-beijing.aliyuncs.com/img/image-20220329184410916.png" alt="image-20220329184410916" style="zoom:80%;" />

```
LSM树的设计思想：将对数据的修改增量保持在内存中，达到指定的大小限制后将修改操作批量写入磁盘。
写入性能大大提升，读取时比较麻烦，需要合并磁盘中历史数据和内存中最近修改操作。

LSM树原理：把一棵大树拆分成N棵小树，首先写入内存中，随着小树越来越大，内存中的小树会flush到磁盘中，磁盘中的树定期可以做merge操作，合并成一棵大树，以优化读性能。

1、小树先写到内存中，为了防止内存数据丢失，写内存的同时需要暂时持久化到磁盘，对应HBase的MemStore和Hlog
2、MemStore上的树达到一定大小后，需要flush到HRegion磁盘中(一般是DN)，这样MemStore就变成了DN上的磁盘文件StoreFile，定期HRegionServer对DN的数据做merge操作，彻底删除无效空间，多棵小树在这个时机合并成大树，增强读性能
```

### <a name="18">HBase的Get和Scan的区别


按指定RowKey获取唯一一条记录，get方法(org.apache.hadoop.hbase.client.Get)

- Get的方法处理分两种：设置了ClosestRowBefore和没有设置的rowlock，主要是用来保证行的事务性

按指定的条件获取一批记录，scan方法(org.apache.hadoop.hbase.client.Scan)

- scan可以通过setCaching与setBatch方法提高速度（以空间换时间）
- scan可以通过setStartRow与setEndRow来限定范围[start，end)，范围越小，性能越高
- scan可以通过setFilter方法添加过滤器，这也是分页，多条件查询的基础。

### <a name="19">HBase和关系型（传统数据库）的区别？


```
1、数据类型：HBase只有简单的数据类型，只保留字符串；传统有丰富的数据类型
2、数据操作：HBase只有简单的增删改查等操作，表和表之间是分离的；传统有各式各样的函数和连接操作
3、存储模型：HBase基于列式存储的，数据即索引，可以实现查询的并发处理；传统基于表格结构和行存储，其没有建立索引将耗费大量的I/O并且建立索引和物化视图需要耗费大量的时间和资源
4、数据维护：HBase的更新实际上是插入了新的数据；传统只是替换和修改
5、可伸缩性：HBase可以轻松的增加或减少硬件的数目，并且对错误的兼容性比较高；传统需要增加中间层才能实现这样的功能
6、事务：HBase只可以实现单行的事务性，意味着行与行、表与表之间不必满足事务性；传统可以实现跨行的事务性
```

