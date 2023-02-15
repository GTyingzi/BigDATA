&emsp;<a href="#0">Spark</a>  
&emsp;&emsp;<a href="#1">Spark的架构</a>  
&emsp;&emsp;<a href="#2">Spark的使用场景</a>  
&emsp;&emsp;<a href="#3">Spark的特点</a>  
&emsp;&emsp;<a href="#4">Spark的四种部署模式</a>  
&emsp;&emsp;<a href="#5">Spark on Yarn模式的优点</a>  
&emsp;&emsp;<a href="#6">Spark任务执行流程(重点)</a>  
&emsp;&emsp;<a href="#7">Spark Shuffle(重点)</a>  
&emsp;&emsp;&emsp;<a href="#8">1、Hash Shuffle</a>  
&emsp;&emsp;&emsp;<a href="#9">2、Sort Shuffle</a>  
&emsp;&emsp;&emsp;<a href="#10">MR、Spark的Shuffle过程对比</a>  
&emsp;&emsp;&emsp;<a href="#11">Shuffle后续优化方向</a>  
&emsp;&emsp;<a href="#12">Spark join</a>  
&emsp;&emsp;&emsp;<a href="#13">1、Broadcast Hash join</a>  
&emsp;&emsp;&emsp;<a href="#14">2、Shuffle Hash join</a>  
&emsp;&emsp;&emsp;<a href="#15">3、Sort-Merge join</a>  
&emsp;&emsp;<a href="#16">Spark的RDD、DataFrame、DataSet、DataStream</a>  
&emsp;&emsp;<a href="#17">RDD的理解</a>  
&emsp;&emsp;<a href="#18">RDD的创建方式</a>  
&emsp;&emsp;<a href="#19">Spark的Job、Stage、Task</a>  
&emsp;&emsp;<a href="#20">Spark容错机制</a>  
&emsp;&emsp;&emsp;<a href="#21">1、宽、窄依赖关系</a>  
&emsp;&emsp;&emsp;<a href="#22">2、RDD Cache缓存</a>  
&emsp;&emsp;&emsp;<a href="#23">3、RDD CheckPoint检查点</a>  
&emsp;&emsp;<a href="#24">Spark的数据本地性</a>  
&emsp;&emsp;<a href="#25">Spark使用parquet文件存储格式的好处</a>  
&emsp;&emsp;<a href="#26">block和parition之间的关系</a>  
&emsp;&emsp;<a href="#27">Unified Memory Management内存管理模型</a>  
&emsp;&emsp;<a href="#28">Spark优化</a>  
&emsp;&emsp;<a href="#29">Spark如何处理不能被序列化的对象</a>  
&emsp;&emsp;<a href="#30">collect的功能</a>  
&emsp;&emsp;<a href="#31">driver的功能</a>  
&emsp;&emsp;<a href="#32">Worker的功能</a>  
## <a name="0">Spark


### <a name="1">Spark的架构


<img src="https://yingziimage.oss-cn-beijing.aliyuncs.com/img/202206211513173.png" alt="image-20220321114714655" style="zoom:80%;" />

```
Spark SQL：提供HQL与Spark进行交互的API，每个数据库表当做一个RDD，Spark SQL查询被转化成Spark操作
Spark Streaming：对实时数据流进行处理和控制，允许程序像普通RDD一样处理实时数据
MLlib：常用的机器学习算法库，实现了对RDD的spark操作
Graphix：控制图、并行图操作和计算的一组算法和工具的集合。扩展了RDD API，包含控制图、创建子图、访问路径上所有顶点操作

Spark Core：包含Spark的基本功能；定义RDD的API、操作。其他Spark的库都是构建在RDD和Spark Core之上
```

### <a name="2">Spark的使用场景


大数据场景主要有以下几种类型：

- 复杂的批处理：偏重点在处于处理海量数据的能力，通常在数十分钟到数小时
- 基于历史数据的交互式查询：通常在数十秒到数十分钟
- 基于实时数据流的大数据处理：通常在数百毫秒到数秒之间

针对以上情况，不用Spark来处理的框架如下：

- 第一种情况用Hadoop的MapReduce来进行海量数据的处理
- 第二章情况用Impala、Kylin进行交互式查询
- 第三种用Storm分布式处理框架处理实时流数据

使用Spark处理：

- 第一张情况用Spark Core
- 第二种情况用Spark SQL
- 第三种情况用Spark Streaming

### <a name="3">Spark的特点


```
快速：与MapReduce相比，Spark基于内存的运算要快100倍以上，基于硬盘而运算也要快10倍以上。
	 实现了高效的DAG执行引擎，计算的中间结果存于内存中，基于内存高效处理数据流
易用：支持Java、Python、Scala等的API，支持80多种高级算法，使用户可以快速构建不同的应用。
	 支持交互式的Python、Scala等的Shell。
通用：可以用于交互查询(Spark SQL)、实时流处理(Spark Streaming)、机器学习(Spark MLllib)、图计算(Graphx)
兼容：非常方便地与其他的开源产品进行融合。Spark可以使用Hadoop的Yarn和Apache Mesos作为它的资源管理和调度器。
	 可以处理所有Hadoop支持的数据，例如HDFS、HBase等。
```

### <a name="4">Spark的四种部署模式


- **本地**：将Spark应用以多线程的方式直接运行在本地，一般是为了方便调试。本地模式分为三类
  - locak：启动一个executor
  - local[k]：启动k个executor
  - local[*]：启动跟CPU数目相同的executor
- **standalone**：分布式部署集群，自带完整的服务，资源、任务监控由spark管理，这个模式也是其他模式的基础
- **Spark on yarn**：分布式部署集群，资源、任务监控由yarn管理，目前仅支持粗粒度资源分配方式，包含cluster、client运行模式
  - cluster适合生产，driver运行在集群子节点，具有容错功能
  - client适合调试，driver运行在客户端
- **Spark On Messos**：官方推荐（血缘关系）。用户可选择两种调度模式运行自己的应用程序
  - 粗粒度模式（Coarse-grained Mode）：每个应用程序的运行环境由一个Driver、若干个Executor组成。其中每个Executor占用若干资源，内部可运行多个Task(对应多少个slot)，应用程序的各个任务正式运行之前，需要将运行环境中的资源全部申请好，且运行过程中要一直占用这些资源，即使不用，最后程序运行结束后回收这些资源
  - 细粒度模式（Fine-grained Mode）：鉴于粗粒度模式会造成大量资源浪费，细粒度模式类似于云计算，按需分配

### <a name="5">Spark on Yarn模式的优点


- 与其他计算框架共享集群资源（Spark框架与MR框架同时运行，若不用Yarn进行资源分配，MR分到的内存资源很少，效率低下）；资源按需分配，进而提高集群资源利用率
- 相较于Spark自带的standalone模式，Yarn的资源分配更加细致
- Application部署简化，例如Spark、Storm等多种框架的应用由客户端提交后，Yarn负责资源的管理和调度，利用Container作为资源隔离的单位，以它为单位去适应内存、CPU等
- Yarn通过队列的方式，管理同时运行在Yarn集群中的多个服务，可根据不同类型的应用程序负载情况，调整对应的资源使用量，实现资源弹性管理



### <a name="6">Spark任务执行流程(重点)


<img src="https://yingziimage.oss-cn-beijing.aliyuncs.com/img/202206211513451.png" alt="image-20220321111821898" style="zoom: 80%;" />

```
1、构建Application的运行环境，Driver创建一个SparkContext
	val conf = new SparkConf().setMaster("local").setAppName("test");
	val sc = new SparkContext(conf);
2、SparkContext向资源管理器(Standalone、Mesos、Yarn)申请Executor资源，资源管理器启动StandaloneExexutorbackend(Executor)
3、Executor向SparkContext申请Task
4、RDD对象建成DAG图，DAG Scheduler将图DAG解析成Stage，每个Stage对应一个Taskset，将其发送给Task Scheduler，由task Scheduler将Task发送给Executor运行
5、Task在Executor上运行，运行完释放所有资源
```

### <a name="7">Spark Shuffle(重点)


导致shuffle操作的几种算子：

- repartition类的操作：repartition、repartitionAndWithPartitons、coalesce等
- byKey类的操作：reduceByKey、groupByKey、SortByKey等
- join类的操作：join、cogroup等

```
repartition类：一般会shuffle，需要对整个集群中，对之前所有的分区数据进行随机、均匀的打乱，然后把数据放入下游新的指定数量的分区内
ByKey类：对一个key进行聚合操作，保证集群中所有节点上相同的key在同一个节点上进行处理
join类：两个rdd进行join，将相同key进行join，shuffle到同一个节点上，再对相同key的两个rdd数据进行笛卡尔
```

Spark Shuffle分为两种：基于Hash的Shuffle、基于Sort的Shuffle

对于下面的内容，先明确一个假设前提：每个Executor只有1个CPU core，也就是说，无论这个Executor上分配多少个task线程，同一时间都只能执行一个task线程。

#### <a name="8">1、Hash Shuffle


未优化的HashShuffle

<img src="https://yingziimage.oss-cn-beijing.aliyuncs.com/img/202206211513777.png" alt="image-20220321121425127" style="zoom:80%;" />

```
当有3个Reducer，各自从Task拉取自己的部分。每个Task将会分3类，这里有4个Task
故总共会生成：4个Task*3个分类文件=12份本地小文件
```

优化后的HashShuffle

<img src="https://yingziimage.oss-cn-beijing.aliyuncs.com/img/202206211513056.png" alt="image-20220321121759066" style="zoom:80%;" />

```
开启合并机制配置：spark.shuffle.consolidateFiles = true(默认false)
在同一个进程中，无论有多少个Task，都将把同样的Key放在同一个Buffer里，然后将Buffer里的数据写入以Core（一个Core只有一种类型的Key）数量为单位的本地文件中。
总共会生成：2个Cores*3个分类文件=6个本地小文件
```

**基于Hash的Shuffle机制的优缺点**

优点：

- 省略不必要的排序开销

缺点：

- 大量小文件的随机读写带来磁盘开销
- 生产文件过多，对文件系统造成压力
- 数据块写入时所需的缓存空间会随之增加，对内存造成压力

#### <a name="9">2、Sort Shuffle


普通Sort Shuffle

<img src="https://yingziimage.oss-cn-beijing.aliyuncs.com/img/202206211513280.png" alt="image-20220321123126397" style="zoom:80%;" />

````
数据会先写入一个数据结构，reduceByKey写入Map，一边通过Map局部融合，一边写入内存。Join算子写入ArrayList到内存，判断是否达到阈值，达到就将数据结构的数据写入磁盘，清空内存数据结构。
在写入磁盘前，先根据key排序，然后分批写入到磁盘文件中。数据会以每批一万条写入磁盘，写入磁盘通过缓冲区溢写的方式，每次溢写都会产生一个磁盘文件，也就是说一个Task过程会产生多个临时文件。
最后在每个Task中，将所有的临时文件合并，写入到最终文件，merge过程。同时单独写一份索引文件，标识下游各个Task的数据在文件中的索引，start offset和end offset
````

bypass Sort Shuffle

<img src="https://yingziimage.oss-cn-beijing.aliyuncs.com/img/202206211513530.png" alt="image-20220321145249526" style="zoom:80%;" />

```
bypass运行机制的触发条件：
	shuffle reduce task数量小于spark.shuffle.sort.bypassMergeThreshold，默认为200
	不是聚合类的shuffle算子，例如reduceByKey

task为每个reduce端的task创建一个临时磁盘文件，数据按key进行hash，将key写入磁盘文件中。写入前先经内存缓存->磁盘文件，最后将所有临时文件合并，创建单独的索引文件
```

Tungsten Sort Shuffle

```
借助Tungsten项目所做的优化来高效处理Shufffle
	实现Tungsten Sort Shuffle机制需要满足的条件：
		Shuffle依赖中不带聚合操作或没有对输出进行排序的要求
		Shuffle的序列化器支持序列化值的重定位
		Shuffle过程的输出分区个数少于16777216个
```

**基于Sort的Shuffle机制的优缺点**

优点：

- 小文件的数量大量减少，Mapper端的内存占用变少
- 在处理大规模的数据上，不会很容易达到性能瓶颈

缺点：

- 强制在Mapper端必须要排序，即使数据本身不需要排序
- 基于记录本身进行排序，这是Sort-Based Shuffle最致命的性能消耗
- 如果Mapper中的Task数量过大，会产生大量小文件，Shuffle传数据到Reducer端时，Reducer对记录进行反序列化，导致大量内存消耗和GC负担巨大。

#### <a name="10">MR、Spark的Shuffle过程对比


|                        | MapReduce                                         | Spark                                                        |
| ---------------------- | ------------------------------------------------- | ------------------------------------------------------------ |
| collect                | 在内存中构造了一块数据结构用于map输出的缓存       | 直接输出写到磁盘文件                                         |
| sort                   | map输出的数据有排序                               | map输出的数据无排序                                          |
| merge                  | 对磁盘上的多个spill文件最后进行合并成一个输出文件 | 在map端没有merge过程，在输出时直接对应一个reduce的数据写到一个文件中，这些文件同时存在并发写，最后不需要合并成一个 |
| copy框架               | jetty                                             | netty或者直接socket流                                        |
| 对于本节点上的文件     | 通过网络框架拉取数据                              | 本地读取                                                     |
| copy过来的数据存放位置 | 先放在内存，内存放不下时写入磁盘                  | 一种方式全部放在内存；另一种先放在内存                       |
| merge sort             | 最后会对磁盘文件、内存中的数据进行合并排序        | 对于采用另一种方式时也有合并排序的过程                       |

#### <a name="11">Shuffle后续优化方向


- 压缩：对数据进行压缩，减少读写数据量
- 内存化：Spark历史版本是这样设计的，Map写数据先把数据全部写入内存中，写完再把数据刷到磁盘上
  - 考虑内存是紧缺资源，修改成把数据直接写入磁盘
  - 具有较大内存的集群，尽量先写入内存，内存放不下了再写入磁盘

### <a name="12">Spark join


hash join是传统数据库中的单机join算法，在分布式环境下需要改造，重复利用分布式计算资源并行计算

```
broadcast hash join：将其中一张小表广播分发到另一张大表所在的分区节点上，分别与其上的分区记录进行hash join

shuffle hash join：小表数据量较大时，采用。根据 join key相同比如分区相同的原理，将两张表分别按照join key进行					重新组织分区，将join分而治之，划分为多个小join，充分利用集群资源并行化
```

#### <a name="13">1、Broadcast Hash join


<img src="https://yingziimage.oss-cn-beijing.aliyuncs.com/img/202206211513409.png" alt="image-20220321154727589" style="zoom:80%;" />

```
broadcast阶段：将小表广播分发到大表所在的所有主机。广播算法有很多，最简单的是先发给driver，driver再统一分发给所有executor；基于bittorrete的p2p2思路

hash join阶段：在每个executor上执行单机版hash join

适用于小表join大表，小表小于 spark.sql.autoBroadcastJoinThreshold，默认为10M
```

#### <a name="14">2、Shuffle Hash join


![](C:\Users\20919\AppData\Roaming\Typora\typora-user-images\image-20220621171715779.png)

```
shuffle阶段：分别将两个表安装join key进行分区，将相同的join key的记录重分布到同一节点，两张表的数据会被重发布到集群中所有节点

hash join阶段：每个分区节点上的数据执行单机版hash join

适用于两张表都足够大，默认设置的shuffle partition的个数是200，将两张表的key基于200做hash取余
```

#### <a name="15">3、Sort-Merge join


<img src="https://yingziimage.oss-cn-beijing.aliyuncs.com/img/202206211513675.png" alt="image-20220321155751745" style="zoom:80%;" />

```
shuuffle阶段：将两张表进行join key重分区，，将相同的join key的记录重分布到同一节点，两张表的数据会被重发布到集群中所有节点

sort阶段：对单个分区节点的两表数据，分别进行排序

merge阶段：对排好序的两张分区表数据执oin操作。遍历两个有序序列，碰到相同join key就merge输出，否则取更小一边。

适用大表join大表，在做等值比较的时候不需要将某一方的全部数据都加载到内存进行计算（升序排序，某个值比它大，后面的值就不用继续比较），在进行等值比较时即用即丢。
```

### <a name="16">Spark的RDD、DataFrame、DataSet、DataStream


```
RDD：分布式数据集，是Spark中最基本的数据抽象。代码中是一个抽象类，代表一个不可变、可分区、里面的元素可并行计算的集合。
DataFrame：是一种以RDD为基础的分布式数据集，类似于传统数据库中的二维表。
DataSet：DataFrame API的一个扩展，是Spark最新的数据抽象，提供RDD的优势以及Spark SQL优化执行引擎的优点
		用户友好的API风格，具有类型安全检查、DataFrame的查询优化特性
		支持编解码器，当需要访问非堆上的数据时可以避免反序列化整个对象
		DataSet是强类型，样例类被定义为数据的结构信息
		DataFram是DataSet的特例，Datarame=Dataset[Row],可以通过as方法将DataFrame转化为DataSet
DataStream：随时间推移而收到数据的序列。在内部，每个时间区间收到的数据都作为RDD存在
```

RDD、DataFrame、DataSet三者的关系

​		在SparkSQL中Spark为我们提供了两个新的抽象，分别是DataFrame和DataSet

Spark1.0	=>	RDD

Spark1.3	=>	DataFrame

Spark1.6	=>	Dataset

​		如果同样的数据都给到这三个数据结构，他们分别计算之后，都会给出相同的结果。不同的是他们的执行效率和执行方式。在后期的Spark版本中，DataSet有可能会逐步取代RDD和DataFrame成为唯一的API接口

**三者的共性**

- 1.全都是Spark平台下的分布式弹性数据集，为处理超大型数据提供便利
- 2.三者都有惰性机制，在进行创建、转换（如map方法）时不会立即执行，只有在遇到Action和foreach时，三者才会开始遍历运算
- 3.有许多共同的函数，如filter,排序等
- 4.在对DataFrame和Dataset进行许多操作时都需要这个包：import spark.implicits._（在创建好SparkSession对象后尽量直接导入）
- 5.都会根据Spark的内存情况自动缓存运算，这样即使数据量很大，也不用担心会内存溢出
- 6.都有partition的概念
- 7.DataFrame和DataSet均可使用模式匹配获取各个字段的值和类型

**三者的区别**

1）RDD

- 一般和spark mllib同时使用
- 不支持sparksql操作

2）DataFrame

- 与RDD和Dataset不同，DataFrame每一行的类型固定为Row，每一列的值没法直接访问，只有通过解析才能获取各个字段的值
- DataFrame与DataSet一般不与spark mllib同时使用
- DataFrame与DataSet均支持SparkSQL的操作，比如select，groupby之类，还能注册临时表/视窗，进行sql语句操作
- DataFrame与DataSet支持一些特别方便的保存方式，比如保存csv，可以带上表头，这样每一列的字段名一目了然

3）DataSet

- Dataset和DataFrame拥有完全相同的成员函数，区别只是每一行的数据类型不同。DataFrame其实就是DataSet的一个特例：tyep DataFrame = Dataset[Row]
- DataFrame也可以叫Dataset[Row]，每一行的类型是Row，不解析，每一行有哪些字段、字段又有什么类型无从得知，只能用上面提到的getAS方法或者共性中的第七条提到的模式匹配拿出特定字段。而DataSet中，每一行是什么类型不一定的，在自定义了case class之后就可以很自由的获得每一行的信息。

### <a name="17">RDD的理解


RDD：弹性分布式数据集，是Spark中最基本的数据处理模型。代码中是一个抽象类，代表一个不可变、可分区、里面的元素可并行计算的集合

```
弹性：
	存储的弹性：内存与磁盘的自动切换
	容错的弹性：数据丢失可以自动恢复
	计算的弹性：计算出错重试机制
	分片的弹性：可根据需要重新分片
	
分布式：数据存储在大数据集群不同节点上

数据集：RDD封装了计算逻辑，并不保存数据

数据抽象：RDD是一个抽象类，需要子类具体实现

不可变：RDD封装了计算逻辑，是不可以改变的，想要改变，只能产生新的RDD，在新的RDD里面封装计算逻辑

可分区、并行计算
```

不足之处

- 不支持细粒度的写和更新操作（如网络爬虫），spark写数据是粗粒度的，所谓粗粒度就是批量写入数据
- 不支持增量迭代计算，Flink支持

### <a name="18">RDD的创建方式


- 使用程序中集合创建
- 使用本地文件系统创建
- 使用hdfs创建
- 基于数据库db创建
- 基于Nosql创建，如HBase
- 基于s3创建
- 基于数据流创建，如socket



### <a name="19">Spark的Job、Stage、Task


一个job含有多个stage，一个stage含有多个task。

```
Job
	每个action的计算会生成一个job，包含很多task的并行计算，可以认为是Spark RDD里面的action
	用户提交的Job会提交给DAGScheduler，被分解为Stage、Task
Stage
	DAGScheduler根据shuffle将job划分为不同的stage，同一个stage包含多个task,这些task有相同的shuffle dependencies
	一个Job被拆分为多组Task，每组Task任务被称为Stage，如：Map Stage、Reduce Stage
Task
	stage下的一个任务执行单元。一般来说，一个RDD有多少个partition，就会有多少个task
	每个executor执行的task的数目，可以由submit，--num-executors(on yarn)来指定
```

### <a name="20">Spark容错机制


```
一般来说，分布式数据集的容错性有两种方式：数据检查点和记录数据的更新

面向大规模数据分析，数据检查点操作成本很高，需要通过数据中心的网络连接在机器之间，复制庞大的数据集。而网络带宽往往比内存带宽低的多，同时还需要消耗更多的存储资源

Spark选择记录数据的更新。如果更新粒度太细太多，那么记录更新成本也不低。因此，RDD只支持粗粒度转换，即只记录单个块上执行的单个操作，然后将创建RDD的一系列变化序列记录下来，以便恢复丢失的分区。

RDD的容错机制又称Lineage（血统）容错，Lineage本质上类似于数据库中的重做日志（Redo Log），只不过此重做日志粒度很大，是对全局数据做同样的重做进而恢复数据
```

Lineage简介

```
	相比其他系统的细颗粒度的内存数据更新级别的备份或者LOG机制，RDD的Lineage记录的是粗颗粒度的特定数据Transformation操作（如filter、map、join等）。当这个RDD的部分分区数据丢失时，它可以通过Lineage获取足够的信息来重新运算和恢复丢失的数据分区。
	这种粗颗粒的数据模型也限制了Spark的运用场合，Spark并不适用于所有高性能要求的场景，但相比细颗粒度模型有性能的提升
```

#### <a name="21">1、宽、窄依赖关系


```
窄依赖：父RDD的一个分区最多被一个子RDD分区所用，有：map、filter等算子
宽依赖：父RDD的一个分区被多个子RDD分区所用，有：groupByKey等算子
```

窄依赖的实现：

```java
abstract class NarrowDependency[T](_rdd: RDD[T]) extends Dependency[T] {
    //返回子RDD的partitionID依赖的所有父RDD的Partition(s)
  def getParents(partitionId: Int): Seq[Int]
  override def rdd: RDD[T] = _rdd
}
```

宽依赖的实现：

```java
class ShuffleDependency[K: ClassTag, V: ClassTag, C: ClassTag](
    @transient private val _rdd: RDD[_ <: Product2[K, V]],
    val partitioner: Partitioner,
    val serializer: Serializer = SparkEnv.get.serializer,
    val keyOrdering: Option[Ordering[K]] = None,
    val aggregator: Option[Aggregator[K, V, C]] = None,
    val mapSideCombine: Boolean = false,
    val shuffleWriterProcessor: ShuffleWriteProcessor = new ShuffleWriteProcessor)
  extends Dependency[Product2[K, V]] {

  if (mapSideCombine) {
    require(aggregator.isDefined, "Map-side combine without Aggregator specified!")
  }
  override def rdd: RDD[Product2[K, V]] = _rdd.asInstanceOf[RDD[Product2[K, V]]]

  private[spark] val keyClassName: String = reflect.classTag[K].runtimeClass.getName
  private[spark] val valueClassName: String = reflect.classTag[V].runtimeClass.getName
  // Note: It's possible that the combiner class tag is null, if the combineByKey
  // methods in PairRDDFunctions are used instead of combineByKeyWithClassTag.
  private[spark] val combinerClassName: Option[String] =
    Option(reflect.classTag[C]).map(_.runtimeClass.getName)

  val shuffleId: Int = _rdd.context.newShuffleId()

  val shuffleHandle: ShuffleHandle = _rdd.context.env.shuffleManager.registerShuffle(
    shuffleId, this)

  _rdd.sparkContext.cleaner.foreach(_.registerShuffleForCleanup(this))
  _rdd.sparkContext.shuffleDriverComponents.registerShuffle(shuffleId)
}
```

#### <a name="22">2、RDD Cache缓存


​		RDD通过Cache或者Persist方法将前面的计算结果缓存，默认情况下会把数据以缓存在JVM的堆内存中。但是并不是这两个方法被调用时立即缓存，而是触发后面的action算子时，该RDD将会被缓存计算节点的内存中，并供后面重用。

![](https://yingziimage.oss-cn-beijing.aliyuncs.com/img/202206211513482.png)

​		缓存有可能丢失，或者存储于内存的数据由于内存不足而被删除，RDD的缓存容错机制保证了即使缓存丢失也能保证计算的正确执行。通过基于RDD的一系列转换，丢失的数据会被重算，由于RDD的各个Partition是相对独立的，因此只需要计算丢失的部分即可，并不需要重算全部Partition

​		Spark会自动对一些Shuffle操作的中间数据做持久化操作（比如：reduceByKey）。这样做的目的是为了当一个节点Shuffle失败了避免重新计算整个输入。但是，在实际使用的时候，如果想重用数据，仍然建议调用persist或cache



#### <a name="23">3、RDD CheckPoint检查点


所谓的检查点其实就是通过将RDD中间结果写入磁盘

由于血缘依赖过长会造成容错成本过高，这样就不如在中间阶段做检查点容错，如果检查点之后有节点出现问题，可以从检查点开始重做血缘，减少了开销。

对RDD进行checkpoint操作并不会马上被执行，必须执行Action操作才能触发。

相关命令：setCheckpointDir

```
缓存Cache和检查点区别Checkpoint

1.Cache缓存只是将数据保存起来，不切断血缘依赖。
  Checkpoint检查点切断血缘依赖
2.Cache缓存的数据通常存储在磁盘、内存等地方，可靠性低。
  Checkpoint的数据通常存储在HDFS等容错、高可用的文件系统，可靠性高。
3.建议对checkpoint()的RDD使用Cache缓存，这样checkpoint的job只需从Cache缓存中读取数据即可，否则需要再从头计算一次RDD
```

### <a name="24">Spark的数据本地性


- PROCESS_LOCAL：读取缓存在本地节点的数据
- NODE_LOCAL：读取本地节点硬盘数据
- ANY：读取非本地节点数据

读数据的规则为：PROCESS_LOCAL > NODE_LOCAL > ANY。PROCESS_LOCAL 和cache有关

### <a name="25">Spark使用parquet文件存储格式的好处


- 如果说HDFS是大数据时代分布式文件系统的首选标准，那么parquet则是整个大数据时代文件存储格式实时首选标准
- **速度更快**：Spark sql操作普通文件csv、parquet文件速度对比上看，绝大多数情况会比使用csv等普通文件速度提升10倍左右，在一些普通文件系统无法在Spark上成功运行的情况下，使用parquet可以运行
- **parquet的压缩技术非常稳定出色**：Spark sql中对压缩技术的处理可能无法正常的完成工作（例如会导致lost task、lost executor），但使用parquet就可以正常的完成
- **优化spark的调度和执行**：有效减少stage的执行消耗、同时可以优化执行路径
- **极大的减少磁盘I/O**：通常情况下能够减少75%的存储空间，由此可以极大的减少Spark sql处理数据时的数据输入内容，尤其在spark1.6x中由下推过滤器情况下能极大减少磁盘IO和内存占用
- **提升扫描吞吐量**：极大提高了数据的查找速度

### <a name="26">block和parition之间的关系


- HDFS中的block是分布式存储的最小单元，等分，可设置冗余。这样设计有一部分磁盘空间的浪费，但是整理的block大小，便于迅速找到、读取对应内容
- Spark中的partion是弹性分布式数据集RDD的最小单元，RDD是由分布在各个节点上的partion组成。partion指spark在计算过程中，生成的数据在计算空间内最小单元，同一份数据(RDD)的partion大小不一，数量不定，根据application里的算子和最初读入的数据分块数量决定
- block位于存储空间，大小固定；partition位于计算空间，大小不固定

### <a name="27">Unified Memory Management内存管理模型


Spark中的内存使用分为两步：执行（execution）、存储（storage）

执行内存主要用于：shuffles、joins、aggregations

存储内存：用于缓存或者跨节点的内部数据传输

Spark1.6之前，一个Executor，内存由以下部分构成：

- **ExecutionMemory**：为了解决shuffles、joins、sorts、aggregations过程中频繁IO需要的buffer，通过spark.shuffle.memoryFraction(默认0.2)配置
- **StorageMemory**：为了解决block cache(显示调用的rdd.cache、rdd.persist等方法)，还有就是broadcasts，以及他上课results的存储，可通过参数spark.storage.memoryFraction(默认0.6)设置
- **OtherMemory**：给系统预留，程序本身运行需要内存(默认0.2)

传统内存管理不足

- Shuffle占用内存0.2*0.8，内存分配这么少，可能会将数据spill到磁盘，频繁的磁盘IO是很大的负担，Storage内存占用0.6，主要是为了迭代处理，传统的Spark内存分配戳操作人的要求非常高（Shuffle分配内存：ShuffleMemoryManager，TaskMemoryManager，ExecutorMemoryManager），一个Task获得全部的Execution的Memory，其他Task过来就没有内存了，只能等待
- 默认情况下，Task在现场中可能会占满整个内存，分片数据

### <a name="28">Spark优化


Spark调优大体上可分为三个方面来进行

- 平台层面：防止不必要的jar包分发，提高数据的本地性，选择高效的存储格式（如：parquet）
- 应用程序层面：过滤操作符的优化降低过多小任务，降低单条记录的资源开销，处理数据倾斜，复用RDD进行缓存，作业并行化执行
- JVM层面调优：设置合适的资源量，设置合理的JVM，启用高效的序列化方法如kyro，增大off head内存等

### <a name="29">Spark如何处理不能被序列化的对象


将不能序列化的内容封装成object

### <a name="30">collect的功能


driver通过collect把集群中各个节点的内容收集过来汇总成结果，抓过来数据是Array型，collect对其进行合并后只有一个元素，tuple类型（KV）

### <a name="31">driver的功能


- 一个Spark左右运行时包括一个Driver进程，也是作业的主进程，具有main函数，并且有SparkContext的实例，时程序的入口点
- 功能：负责向集群申请资源，向master注册信息，负责作业调度、作业解析、生成Staeg并调度Task到Executor上，包括DAGScheduler、TaskScheduler

### <a name="32">Worker的功能


**功能**：管理当前节点内存，CPU的使用状况，接收master分配过来的资源指令，通过ExecutorRunner启动程序分配任务，worker就类似于包工头， 管理分配新进程，做计算的服务，相当于process服务

注意：

- worker会不会汇报当前信息给master，worker心跳给master主要只有workid，它不会发送资源信息以心跳的方式给mater，master分配的时候就知道work， 只有出现故障的时候才会发送资源
- worker不会运行代码，具体运行的是Executor是可以运行具体appliaction写的业务逻辑代码，操作代码的节点，它不会运行程序的代码的

