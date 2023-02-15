&emsp;<a href="#0">Kylin</a>  
&emsp;&emsp;<a href="#1">1.Kylin简介</a>  
&emsp;&emsp;&emsp;<a href="#2">1.1Kylin定义</a>  
&emsp;&emsp;&emsp;<a href="#3">1.2Kylin架构</a>  
&emsp;&emsp;&emsp;<a href="#4">1.3Kylin特点</a>  
&emsp;&emsp;<a href="#5">2.维度和度量</a>  
&emsp;&emsp;&emsp;<a href="#6">2.1维度和度量</a>  
&emsp;&emsp;&emsp;<a href="#7">2.2Cube和Cuboid</a>  
&emsp;&emsp;&emsp;<a href="#8">2.3Cube构建算法</a>  
&emsp;&emsp;&emsp;&emsp;<a href="#9">1）逐层构建算法（layer）</a>  
&emsp;&emsp;&emsp;&emsp;<a href="#10">2）快速构建算法（inmem）</a>  
&emsp;&emsp;&emsp;<a href="#11">2.4.4Cube存储原理</a>  
&emsp;&emsp;<a href="#12">3.Kylin Cube构建优化</a>  
&emsp;&emsp;&emsp;<a href="#13">3.1使用衍生维度（derived dimension）</a>  
&emsp;&emsp;&emsp;<a href="#14">3.2使用聚合组（Aggregation group）</a>  
&emsp;&emsp;&emsp;<a href="#15">3.3Row Key优化</a>  
## <a name="0">Kylin


### <a name="1">1.Kylin简介


#### <a name="2">1.1Kylin定义


​		Apache Kylin是一个开源的分布式分析引擎，提供Haddop/Spark之上的SQL查询接口及多维分析（OLAP）能力以及支持大规模数据，最初由eBay Inc开发并贡献至开源社区。它能在亚秒内查询巨大的Hive表

#### <a name="3">1.2Kylin架构

![](https://yingziimage.oss-cn-beijing.aliyuncs.com/img/202302151602000.png)
```
1)REST Server(REST 服务层)
	REST Server是一套面向应用程序开发的入口点，旨在实现针对Kylin平台的应用开发工作。此类应用程序可以提供查询、获取结果、触发cube构建任务、获取元数据以及获取用户权限等，另外可以通过Restful接口实现SQL查询
	
2)Query Engine(查询引擎)
	当cube准备就绪后，查询引擎就能够获取并解析用户查询，它随后会与系统中的其他组件进行交互，从而向用户返回对应的结果
	
3)Routing(路由器)
	在最初设计时曾考虑过将Kylin不能执行的查询引导去Hive中继续执行，但在实践后发现Hive与Kylin的速度差异过大，导致用户无法对查询的速度有一致的期望，很可能大多数查询几秒内就返回结果了，而有些查询则要等几分钟到几十分钟，因此体验非常糟糕。最后这个路由功能在发行版中默认关闭。
	
4)Metadata(元数据管理)
	Kylin是一款元数据驱动型应用程序。元数据管理工具是一大关键性组件，用于对保存在Kylin当中的所有所有元数据进行管理，其中包括最为重要的cube元数据。其他全部组件的正常运作都需要以元数据管理工具为基础。Kylin的元数据存储在hbase中
	
5)Cube Build Engine(任务引擎)
	这套引擎的设计目的在于处理所有离线任务，其中包括shell脚本、java API以及MapReduce任务等等。任务引擎对Kylin当中的全部任务加以管理与协调，从而确保每一项任务都能得到切实执行并解决期间出现的故障。
```

#### <a name="4">1.3Kylin特点


​		Kylin的主要特点包括支持SQL接口、支持超大规模数据集、亚秒级响应、可伸缩性、高吞吐率、BI工具集成等。

````
1）标准SQL接口：Kylin是以标准的SQL作为对外服务的接口
2）支持超大数据集：Kylin对于大数据的支撑能力可能是目前所有技术中最为领先的。早在2015年eBay的生产环境中就能支持百亿记录的秒级查询，之后在移动的应用场景中又有千亿记录秒级查询的按理。
3）亚秒级响应：Kylin拥有优异的查询相应速度，这点得益于预计算，很多复杂的计算，比如连接、聚合，在离线的预计算过程中就已经完成，这大大降低了查询时刻所需的计算量，提高了响应速度。
4）可伸缩性和高吞吐率：单节点Kylin可实现每秒70个查询，还可以搭建Kylin的集群，
5）BI工具集成：BI可以与现有的BI工具集成，具体内容包括如下内容
	ODBC:与Tableau、Excel、PowerBI等工具集成
	JDBC:与Saiku、BIRT等Java工具集成
	RestAPI:与JavaScript、Web网页集成
Kylin开发团队还提供了Zepplin插件，可以使用Zepplin访问Kylin服务
````

### <a name="5">2.维度和度量


#### <a name="6">2.1维度和度量


​		维度：即观察数据的角度。比如员工数据，可以从性别角度来分析，也可以更加细化，从入职时间或者地区的维度来观察。维度是一组离散的值，比如说性别中的男和女，或者时间维度上的每一个独立的日期。因此在统计时可以将维度值相同的记录聚合在一起，然后应用聚合函数做累加、平均、最大和最小值等聚合计算。

​		度量：即被聚合（观察）的统计值，也就是聚合运算的结果。比如说员工数据中不同性别员工的人数，又或者说在同一年入职的员工有多少

#### <a name="7">2.2Cube和Cuboid


​		有了维度跟度量，一个数据表或者数据模型上的所有字段就可以分类了，它们要么是维度，要么是度量（可以被聚合）。于是就有了根据维度和度量做预计算的Cube理论。

​		给定一个数据模型，我可以对其上的所有维度进行聚合，对于N个维度来说，组合的所有可能性共有2^n种。对于每一种维度的组合，将度量值做聚合计算，然后将结果保存为一个物化视图，称为Cuboid。所有维度组合的Cuboid作为一个整体，称为Cube。

​		下面举一个简单的例子说明，假设有一个电商的销售数据集，其中维度包括时间[time]、商品[item]、地区[location]和供应商[supplier]，度量为销售额。那么所有维度的组合就有2^4=16种，如下所示：
![](https://yingziimage.oss-cn-beijing.aliyuncs.com/img/202302151602974.png)
一维度的组合有C_4^1=4种

二维度的组合有C_4^2=6种

三维度的组合有C_4^3=4种

零维度+四维度各一种，总共16种

#### <a name="8">2.3Cube构建算法


##### <a name="9">1）逐层构建算法（layer）

![](https://yingziimage.oss-cn-beijing.aliyuncs.com/img/202302151602086.png)
一个N维的Cube,由2^N个子立方体组成。

在逐层算法中，按维度数逐层减少来计算，每个层级的计算（除了第一层，它是从原始数据聚合而来），是基于它上一层级的结果来计算的。比如[group by A,B]的结果，可以基于[Group by A,B,C]的结果，通过去掉C后聚合得来的；这样可以减少重复计算；当0维度Cuboid计算出来的时候，整个Cube的计算也就完成了。

​		每一轮的计算都是一个MapReduce任务，且串行执行；一个N维的Cube，至少需要N次MapReduce Job
![](https://yingziimage.oss-cn-beijing.aliyuncs.com/img/202302151603625.png)

```
算法优点：
	1）此算法充分利用了MapReduce的优点，处理了中间复杂的排序和shuffle工作，故而算法代码清晰简单，易于维护
	2）受益于Hadoop的日趋成熟，此算法非常稳定，即便是集群资源紧张时，也能保证最终能够完成
	
算法缺点：
	1）当Cube有比较多维度的时候，所需要的MapReduce任务也相应增加；由于Hadoop的任务调度需要耗费额外资源，特别是集群庞大的时候，反复递交任务造成的额外开销会相当可观；
	2）由于Mapper逻辑中并未进行聚合操作，所以每轮MR的shuffle的工作量都很大，导致效率低下。
	3）对HDFS的读写操作较多：每一层计算的输出会用做下一层计算的输入，这些Key-Value需要写到HDFS上；当所有计算都完成后，Kylin还需要额外的一轮任务将这些文件转成HBase的HFile格式，以导入到HBase中去
	总体而言，该算法的效率较低，尤其是当Cube维度数较大的时候
```

##### <a name="10">2）快速构建算法（inmem）

![](https://yingziimage.oss-cn-beijing.aliyuncs.com/img/202302151603889.png)
		也被称作逐段（By Segment）或逐块（By Split）算法，从1.5x开始引入该算法，该算法的主要思想是，每个Mapper将其所分配到的数据块，计算成一个完整的小Cube段（包含所有Cuboid).每个Mapper将计算完的Cube段输出个Reducer做合并，生成大Cube，也就是最终结果。如下图所示
![](https://yingziimage.oss-cn-beijing.aliyuncs.com/img/202302151603754.png)
与旧算法相比，快速算法主要有两点不同

```
1）Mapper会利用内存做预聚合，算出所有；Mapper输出的每个Key都是不同的，这样会减少输出到Hadoop MapReduce的数据量，Combiner也不再需要。
2）一轮MapReduce便会完成所有层次的计算，减少Hadoop任务的调配
```
#### <a name="11">2.4.4Cube存储原理

![](https://yingziimage.oss-cn-beijing.aliyuncs.com/img/202302151603940.png)
![](https://yingziimage.oss-cn-beijing.aliyuncs.com/img/202302151603761.png)

### <a name="12">3.Kylin Cube构建优化


#### <a name="13">3.1使用衍生维度（derived dimension）


​		衍生维度用于在有效维度内将维度表上的非主键维度排除掉，并使用维度表的主键（其实是事实上相应的外键）来替代它们。Kylin会在底层记录维度表主键与维度表其他维度之间的映射关系，以便在查询时能够动态地维度表的主键“翻译”成这些非主键维度，并进行实时聚合。
![](https://yingziimage.oss-cn-beijing.aliyuncs.com/img/202302151604424.png)
​		虽然衍生维度具有非常大的吸引力，但这也并不说所有维度表上的维度都得变成衍生维度，如果从维度表主键到某个维度表维度所需要的聚合工作量非常大，则不建议使用衍生维度。

#### <a name="14">3.2使用聚合组（Aggregation group）


​		聚合组是一种强大的剪枝工具。

​		对于每个分组内部的维度，有三种方式定义。

1）强制维度
![](https://yingziimage.oss-cn-beijing.aliyuncs.com/img/202302151604019.png)
2）层级维度
![](https://yingziimage.oss-cn-beijing.aliyuncs.com/img/202302151604873.png)
3）联合维度
![](https://yingziimage.oss-cn-beijing.aliyuncs.com/img/202302151604826.png)

#### <a name="15">3.3Row Key优化


​		Kylin会把所有的维度按照顺序组合成一个完整的Rowkey，并且按照这个Rowkey升序排列Cuboid中所有的行。

​		设计良好的Rowkey将更有效地完成数据的查询过滤和定位，减少IO次数，提高查询速度，维度在rowkey中的次序，对查询性能有显著的影响。

​		Row key的设计原则如下：

1）被用作过滤的维度放在前边
![](https://yingziimage.oss-cn-beijing.aliyuncs.com/img/202302151604378.png)
2）基数大的维度放在基数小的维度前边
![](https://yingziimage.oss-cn-beijing.aliyuncs.com/img/202302151605500.png)

