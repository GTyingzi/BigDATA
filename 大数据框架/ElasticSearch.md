&emsp;<a href="#0">ElasticSearch</a>  
&emsp;&emsp;<a href="#1">ES基础</a>  
&emsp;&emsp;<a href="#2">数据格式</a>  
&emsp;&emsp;<a href="#3">倒排索引</a>  
&emsp;&emsp;<a href="#4">DocValues</a>  
&emsp;&emsp;<a href="#5">text和keyword类型的区别</a>  
&emsp;&emsp;<a href="#6">什么是停顿词过滤</a>  
&emsp;&emsp;<a href="#7">query和filter的区别</a>  
&emsp;&emsp;<a href="#8">ES写入流程：</a>  
&emsp;&emsp;&emsp;<a href="#9">es写数据过程</a>  
&emsp;&emsp;&emsp;<a href="#10">es写数据底层</a>  
&emsp;&emsp;<a href="#11">ES的更新和删除流程</a>  
&emsp;&emsp;<a href="#12">ES的搜索流程</a>  
&emsp;&emsp;<a href="#13">ES在高并发下保证读写一致性</a>  
&emsp;&emsp;<a href="#14">ES选举Master节点</a>  
&emsp;&emsp;&emsp;<a href="#15">1、ES的分布式原理</a>  
&emsp;&emsp;&emsp;<a href="#16">2、ES如何选举Master</a>  
&emsp;&emsp;&emsp;<a href="#17">3、ES是如何避免脑裂现象</a>  
&emsp;&emsp;<a href="#18">建立索引阶段性能提升方法</a>  
&emsp;&emsp;<a href="#19">ES的深度分页与滚动搜索</a>  
&emsp;&emsp;&emsp;<a href="#20">深度分页</a>  
&emsp;&emsp;&emsp;<a href="#21">滚动搜索</a>  
&emsp;&emsp;<a href="#22">ES对于大数据量的聚合如何实现</a>  
## <a name="0">ElasticSearch


### <a name="1">ES基础


ELastic Stack（ELK Stack）：包括Elasticsearch、Kibana、Beats和Logstash

能够安全可靠地获取任何源、任何格式的数据，然后实时地对数据进行搜索、分析和可视化

Elasticsearch（ES）是一个开源的高扩展的分布式全文搜索引擎，是整个Elastic Stack技术栈的核心。它可以近乎实时的存储、检索数据；本身扩展性很好，可以扩展到百台服务器，处理PB级别数据

### <a name="2">数据格式


ES是面向文档型数据库，一条数据在这里就是一个文档。ES存储文档数据和关系型数据库MySQL存储数据的概念进行一个类比

![](https://yingziimage.oss-cn-beijing.aliyuncs.com/img/image-20220618113556305.png)

- index（索引）：类似于MySQL中的数据库，ES中的索引就是存放数据的地方，包含了一堆相似结构的文档数据
- type（类型）：定义数据结构，可以认为是MySQL中的一张表，type是index中的一个逻辑数据分类
- document（文档）：类似于MySQL中的一行，不同之处在于ES中的每个文档可以有不同的字段，但对于通用字段应该具有相同的数据类型，文档是ES中的最小数据单元，可以认为文档就是一条记录
- Field字段：ES的最小单位，一个document里面有多个field

分布式说明：

- **shard**（分片）：单台机器无法存储大量数据，ES可以将一个索引中的数据切分为多个shard，分布在多台服务器上存储。有了shard就可以横向扩展，存储更多数据，让搜索和分析等操作分布到多台服务器上执行，提升吞吐量性能
  - primary shard（主分片）：建立索引时一次设置，不能修改，默认为5个
  - replica shard（副分片）：默认1个，可随时修改
  - 默认每个索引10个shard，即5个primary shard、5个replica shard。最小的高可用配置为 2 台服务器
- **replica**（副本）：任何一个服务器随时可能故障或宕机，此时shard可能会丢失，因此可以为每个shard创建多个replica副本。replica可以在shard故障时提供备用服务，保证数据不丢失，多个replica还可以提升搜索操作的吞吐量和性能

用JSON作为文档序列化的格式，比如一条用户信息

```json
{
 "name" : "John",
 "sex" : "Male",
 "age" : 25,
 "birthDate": "1990/05/01",
 "about" : "I love to go rock climbing",
 "interests": [ "sports", "music" ]
}
```

### <a name="3">倒排索引


在搜索引擎中，每个文档都有一个对应的文档ID，文档内容表示为一系列关键词的集合。例如，某个文档经过分词，提取20个关键词，每个关键词都会记录它在文档中出现的次数和出现的位置

倒排索引：关键词到文档ID的映射。每个关键词都对应这一系列的文件，这些文件中都出现了该关键词，有了倒排索引，搜索引擎可以很方便地响应用户查询

- 倒排索引中的所有词项对应一个或多个文档
- 倒排索引中的词项根据字典顺序升序排列

### <a name="4">DocValues


倒排索引也有缺陷，当我们需要对数据进行聚会操作（如排序、分组等），lucene内部会遍历提取所有出现在文档集合的排序字段，然后再次构建一个最终的排好序的文档集合list，这个步骤的过程全部维持在内存中操作，若排序数据量巨大，非常容易造成solr内存溢出和性能缓慢

DocValues就是ES在构建倒排索引的同时，构建了正排索引，保持了docld到各个字段值的映射，可以看作是以文档为维度，从而实现根据指定字段进行排序和聚合的功能

DocValues保存于操作系统的磁盘中

- 当DocValues远小于节点的可用内存，操作系统将所有DocValues存于内存中（堆外内存），有助于快速访问
- 当docValues大于节点的可用内存，ES可从操作系统页缓存中加载或弹出，从而避免发生内存溢出的异常

### <a name="5">text和keyword类型的区别


分词的区别：

- text类型存入ES时，会先分词，然后根据分词后的内容建立倒排索引
- keyword类型不会分词，直接根据字符串内存建立倒排索引，keyword类型的字段只能通过精确值搜索到

### <a name="6">什么是停顿词过滤


没有意义的词，比如“的”、“而”等，这类词没必要建立索引

### <a name="7">query和filter的区别


- query：查询操作不仅仅会进行查询，还会计算分值，用于确定相关度
- filter：查询操作仅判断是否满足查询条件，不会计算任何分值，也不会关心返回的排序问题。同时filter查询的结构可以被缓存，提高性能

### <a name="8">ES写入流程：


#### <a name="9">es写数据过程


![](https://yingziimage.oss-cn-beijing.aliyuncs.com/img/202206222307361.png)

1. 客户端选择一个node发送请求，该node为coordinating node（协调节点）
2. coordinating node对document进行路由，将请求转发给响应的node（有primary shard）
3. 实际的node上的primary shard处理请求，然后将数据同步到replica node
4. coordinating node等到primary node和所有replica node 都执行成功后，就返回响应结果给客户端

#### <a name="10">es写数据底层


![](https://yingziimage.oss-cn-beijing.aliyuncs.com/img/202206222313580.png)

1. 数据先写入memory buffer，然后定时（默认每隔1s）将memory buffer中的数据写入一个新的segment文件中，并进入Filesystem cache（同时清空memory buffer），这个过程称为refresh
   - **ES的近时性**：数据存在memory buffer时搜索不到，只有数据被refresh到Filesystem cache之后才能被搜索到，而refresh是每秒一次，所以称es是近实时的，可以通过手动调用es的api触发refresh操作，让数据马上可以搜索到
2. memory buffer和Filesystem Cache都是基于内存，假设服务器宕机，那么数据就会丢失，故ES通过Transaction log日志文件来保证数据的可靠性，在数据写入memory buffer的同时，将数据写入Transaction log日志文件中，在机器宕机重启时，es会自动读取Transaction log日志文件中的数据，恢复到memory buffer和Filesystem cache中去
   - **ES数据丢失问题**：Transaction log也是先写入Filesystem cache，然后默认每隔5s秒刷一次到磁盘中。在默认情况下，可能有5秒的数据会仅仅停留在memory buffer或者Transaction log文件的Filesystem cache中，而不在磁盘上，若此时机器宕机，会丢失5秒的数据，可以将Transaction log文件设置成每次写操作必须是直接fsync到磁盘，但性能会降低
3. flush操作：不断重复上面步骤，Transaction log会变得越来越大，文件超过512M或每30分钟后，就会触发commit操作，即flush操作
   - 将buffer中的数据refresh到Filesystem Cache中去，清空buffer
   - 创建一个新的commit point（提交点），同时强行将Filesystem Cache中目前所有的数据都fsync到磁盘文件中
   - 删除旧的Transaction log日志文件，并创建一个新的Transaction log日志文件，此时commit操作完成

### <a name="11">ES的更新和删除流程


删除和更新都是写操作，但由于ES中的文档都是不可变的，因此不能被删除或者改动以展示其变更，ES利用.del文件标记文档是否被删除，磁盘上的每个段都有一个相应的.del文件

- 删除操作：文档没有真被删除，在.del文件中被标为deleted状态，该文档依然能匹配查询，但是在结果中被过滤
- 更新操作：将旧的doc标记为deleted状态，然后创建一个新的doc

memory buffer每refresh一次，就会产生一个segment文件，故默认1s生成一个segment文件。每次merge会将多个segment文件合并成一个，同时这里会将标识为deleted的doc给物理删除掉，不写入到新的segment中，然后将新的segment文件写入磁盘，这里会写一个commit point，标识所有新的segment文件，然后打开segment文件供搜索使用，同时删除旧的segment文件

### <a name="12">ES的搜索流程


搜索被执行成一个两阶段过程：Query、Fetch

- Query：客户端发送请求到协调节点(coordinate node)，协调节点将搜索请求广播到所有primary shard或replica shard。每个分片在本地执行搜索并构建一个匹配文档大小为from + size的优先队列。每个分片返回各自优先队列中所有文档的ID和排序值给协调节点，由协调节点及逆行数据的合并、排序、分页等操作，产出最终结果
- Fetch：协调节点根据doc id去各个节点上查询实际的document数据，由协调节点返回结果给客户端
  - 协调节点对doc id进行哈希路由，将请求转发到对应的node，此时会使用round-robin随机轮询算法，在primary shard以及其所有replica中随机选择一个，让读请求负载均衡
  - 接收请求的node返回document给协调节点
  - 协调节点返回document给客户端

Query Then Fetch的搜索类型在文档相关性打分的时候参考的时本分片的数据，在文档数量较少时可能不够准确，DFS Query Then Fetch增加了一个预查询的处理，询问Term和Document frequency，这个评分更准确，但性能会变差

### <a name="13">ES在高并发下保证读写一致性


**更新操作**：可以通过版本号来使用乐观并发控制，以确保新版本不会被旧版本覆盖

```
每个文档都由一个_version版本号，这个版本号在文档被改变时+1，ES使用这个_version保证所有修改都被正确排序，当一个旧版本出现在新版本之后，它会被简单的忽略
利用_version的这一优点确保数据不会因为修改冲突而丢失，比如指定文档的version来做更改，如果那个版本号不是现在，就请求失败
```

**写操作**：一致性级别支持quorum/one/all，默认为quorum，即只有当大多数分片可用时才允许写操作，但即使大多数可用，也可能存在网络等原因导致写入副本失败，这样的副本会被认为故障，分片将会在一个不同的节点上重建

- one：只要有一个primary shard是active活跃可用，就可以执行
- all：必须所有的primary shard和replica shard都是活跃的，才可以执行这个写操作
- quorum：默认的值，要求所有的shard中，必须大部分的shard都是活跃的，可用的，才可以执行这个写操作

**读操作**：可设置为replication为sync（默认），这使得操作在主分片和副本分片都完成后才会返回；若设置replication为async时，也可以通过设置搜索请求参数\_prference为primary来查询主分片，确保文档是最新版本

### <a name="14">ES选举Master节点


#### <a name="15">1、ES的分布式原理


```
ES会对存储的数据进行切分，将数据划分到不同的分片上，同时每一个分片会保存多个副本，主要是为了保证分布式环境的高可用。在ES中，节点是对等的，节点间会选取集群的Master，由Master会负责集群状态信息的改变，并同步给其他节点

ES的性能不会很低：只有建立索引和类型需要经过Master，数据的写入有一个简单的Routing规则，可以路由到集群中的任意节点，所以数据写入压力是分散在整个集群的
```

#### <a name="16">2、ES如何选举Master


```
ES的选举由ZenDiscovery模块负责，主要包含Ping（节点间通过这个RPC来发现彼此）、Unicast（单播模块包含一个主机列表以控制哪些节点需要ping通）
-确认候选主节点的最少投票通过数量，elasticsearch.yml设置的值discovery.zen.minimum_master_nodes
-对所有候选master的节点（node.master:true）根据nodeId字典排序，每次选举每个节点都将把自己知道的节点拍一次序，然后选出第一个（0位）节点，暂且认为它是master节点
-若对某个节点的投票数达到阈值，并且该节点自己也选举自己，那该节点就是master,否则重新选举一直到满足上述条件

补充：master节点的职责主要包括集群、节点、索引的管理，不负责文档级别的管理；data节点可以关闭http功能
```

#### <a name="17">3、ES是如何避免脑裂现象


```
(1)当集群中master候选节点数量不小于3个时（node.master: true），可以通过设置最少投票通过数量（discovery.zen.minimun_master_nodes），设置超过所有候选节点一半以上来解决脑裂问题，即设置为 N/2 + 1

（2）当集群master候选节点只有两个时，这种情况是不合理的，最好把另一个node.master改成false，不然两个master备选节点只要有一个挂了，就选不出master了
```

### <a name="18">建立索引阶段性能提升方法


- 使用SSD存储介质
- 使用批量请求并调整其大小：每次批量数据5-15MB
- 在做大批量导入，考虑通过设置：index.number_of_replicas：0关闭副本
- 若你的搜索结果不需要近实时的准确度，考虑把每个索引的 index.refresh_interval改到30s
- 段和合并：ES默认值是20MB/s，但若使用SSD，可以考虑提高到100-200MB/s，若在做批量导入，完全不在意搜索，可以彻底关掉合并限流
- 增加 index.translog.flush_threshold_size设置，从默认的512MB到更大一些的值，比如1GB

### <a name="19">ES的深度分页与滚动搜索


#### <a name="20">深度分页


```
深度分页其实就是搜索的深浅度，比如第1页、第2也..第100页等，是比较浅的；第10000页...之类的就很深，探索的太深就会造成性能问题，耗费内存和占用CPU，而且ES为了性能，他不支持超过一万条数据以上的分页查询

如何解决深度分页带来的问题？
我们应该避免深度分页操作（限制分页页数），比如最多只能提供100页的展示，那么101页开始就没了，毕竟用户也不会搜的那么深
```

#### <a name="21">滚动搜索


```
一次性查询1万+数据，往往会造成性能影响，这个时候可以使用滚动搜索，也就是scroll。滚动搜索可以先查询一些数据，然后再紧接着以此往下查询
在第一次查询的时候会有一个滚动id，相当于锚标记，随后再次滚动搜索会需要上一次搜索滚动id，根据这个进行下一次的搜索请求。每次搜索都是基于一个历史数据快照，查询数据的期间，若有数据变更和搜索无关
```

### <a name="22">ES对于大数据量的聚合如何实现


ES提供的首个近似聚合是cardinality度量。它提供一个字段的基数，即该字段的distinct或者unique值的数目

它是基于HLL算法，HHL会先对我们的输入作哈希运算，然后根据哈希运算的结果中的bits做概率估算从而得到基数，特点为

- 可配置的精度，用来控制内存的使用（更精确 = 更多内存）
- 小的数据集精度是非常高的
- 可通过配置参数，去设置需要的固定内存使用量。无论数千还是数十亿的唯一值，内存使用量只与你配置的精确度相关

**科普&拓展**

```
HyperLogLog：
下面简称为HLL，它是 LogLog 算法的升级版，作用是能够提供不精确的去重计数。存在以下的特点：
1. 能够使用极少的内存来统计巨量的数据，在 Redis 中实现的 HyperLogLog，只需要12K内存就能统计2^64个数据。
2. 计数存在一定的误差，误差率整体较低。标准误差为 0.81% 。
3. 误差可以被设置辅助计算因子进行降低。
---
应用场景：
1. 基数不大，数据量不大就用不上，会有点大材小用浪费空间
2. 有局限性，就是只能统计基数数量，而没办法去知道具体的内容是什么
3. 和bitmap相比，属于两种特定统计情况，简单来说，HyperLogLog 去重比 bitmap 方便很多
4. 一般可以bitmap和hyperloglog配合使用，bitmap标识哪些用户活跃，hyperloglog计数
---
应用场景：
1. 基数不大，数据量不大就用不上，会有点大材小用浪费空间
2. 有局限性，就是只能统计基数数量，而没办法去知道具体的内容是什么
3. 和bitmap相比，属于两种特定统计情况，简单来说，HyperLogLog 去重比 bitmap 方便很多
4. 一般可以bitmap和hyperloglog配合使用，bitmap标识哪些用户活跃，hyperloglog计数
```

