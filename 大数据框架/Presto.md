&emsp;<a href="#0">Presto</a>  
&emsp;&emsp;<a href="#1">1.Presto简介</a>  
&emsp;&emsp;&emsp;<a href="#2">1.1Presto概念</a>  
&emsp;&emsp;&emsp;<a href="#3">1.2Presto架构</a>  
&emsp;&emsp;&emsp;<a href="#4">1.3Presto优缺点</a>  
&emsp;&emsp;<a href="#5">2.Presto安装</a>  
&emsp;&emsp;&emsp;<a href="#6">2.2Presto命令行Client安装</a>  
&emsp;&emsp;&emsp;<a href="#7">2.3Presto可视化Client安装</a>  
&emsp;&emsp;<a href="#8">3.Presto优化之数据存储</a>  
&emsp;&emsp;&emsp;<a href="#9">3.1合理设置分区</a>  
&emsp;&emsp;&emsp;<a href="#10">3.2使用列式存数</a>  
&emsp;&emsp;&emsp;<a href="#11">3.3使用压缩</a>  
&emsp;&emsp;<a href="#12">4.Persto优化之查询SQL</a>  
&emsp;&emsp;&emsp;<a href="#13">4.1只选择使用的字段</a>  
&emsp;&emsp;&emsp;<a href="#14">4.2过滤条件必须加上分区字段</a>  
&emsp;&emsp;&emsp;<a href="#15">4.3Group By语句优化</a>  
&emsp;&emsp;&emsp;<a href="#16">4.4Order by使用Limit</a>  
&emsp;&emsp;&emsp;<a href="#17">4.5使用Join语句时将大表放在左边</a>  
&emsp;&emsp;<a href="#18">5.注意事项</a>  
&emsp;&emsp;&emsp;<a href="#19">5.1字段名引用</a>  
&emsp;&emsp;&emsp;<a href="#20">5.2时间函数</a>  
&emsp;&emsp;&emsp;<a href="#21">5.3不支持INSERT OVERWRITE语法</a>  
&emsp;&emsp;&emsp;<a href="#22">5.4PARQUET格式</a>  
## <a name="0">Presto


### <a name="1">1.Presto简介


#### <a name="2">1.1Presto概念


​		Presto是一个开源的分布式SQL查询引擎，数据量支持GB到PB字节，主要用来处理秒级查询的场景

注：虽然Presto可以解析SQL，但它不是一个标准的数据库。不是MySQL、Oracle的代替品，也不能用来处理在线事务（OLTP）

#### <a name="3">1.2Presto架构


Presto由一个Coordinator和多个Worker组成
![](https://yingziimage.oss-cn-beijing.aliyuncs.com/img/202302151556290.png)

#### <a name="4">1.3Presto优缺点

![](https://yingziimage.oss-cn-beijing.aliyuncs.com/img/202302151556248.png)
```
优点：
（1）Presto基于内存运算，减少硬盘IO，计算更快
（2）能够连接多个数据源，跨数据源连表查看，如从Hive查询大量网站访问记录，然后从Mysql中匹配出设备信息

缺点：
Pressto能够处理PB级别的海量数据分析，但Presto并不是把PB级数据都存放在内存中计算的，而是根据场景。
	Count,AVG等聚合运算，是边读数据边计算，再清内存，再读数据，再计算，这种耗的内存并不高。
	连表查询，可能产生大量的临时数据，速度会变慢
```

### <a name="5">2.Presto安装


2.1Presto Server安装

1）官网地址

https://prestodb.github.io/

2）下载地址

https://repo1.maven.org/maven2/com/facebook/presto/presto-server/0.208/presto-server-0.208.tar.gz

3）解压到/opt/module

```
tar -zxvf presto-server-0.208.tar.gz -C /opt/module/
```

4）将文件更名为presto 并在presto目录下创建data,etc等文件夹

```
mv presto-server-0.208/ presto
...
cd presto
mkdir data
mkdir etc
```

5）进入etc配置jvm.config

```
vim jvm.config

-server
-Xmx16G
-XX:+UseG1GC
-XX:G1HeapRegionSize=32M
-XX:+UseGCOverheadLimit
-XX:+ExplicitGCInvokesConcurrent
-XX:+HeapDumpOnOutOfMemoryError
-XX:+ExitOnOutOfMemoryError
```

6）配置catalog数据源

在etc下创建catalog，进入catalog,此处我们可配置一个Hive的catalog

```
vim hive.properties

connector.name=hive-hadoop2
hive.metastore.uri=thrift://hadoop102:9083
```

7）将presto分发给hadoop103、hadoop104

```
xsync /opt/module/presto
```

8）修改hadoop102、hadoop103、hadoop104三台主机的/opt/module/presto/etc/node.properties

使得三个的node.id各不相同

9）Presto由一个coordinator节点和多个worker节点组成

在hadoop102上配置coordinator

```
etc目录下 vim config.properties

coordinator=true
node-scheduler.include-coordinator=false
http-server.http.port=8881
query.max-memory=50GB
discovery-server.enabled=true
discovery.uri=http://hadoop102:8881
```

在hadoop103、hadoop104上配置worker

```
etc目录下 vim config.properties

coordinator=false
http-server.http.port=8881
query.max-memory=50GB
discovery.uri=http://hadoop102:8881
```

10）在hadoop102的/opt/module/hive目录下，启动Hive Metastore，用yingzi角色

```
nohup bin/hive --service metastore >/dev/null 2>&1 &
```

11）分别在hadoop102、hadoop103、hadoop104上启动Presto Server



```
(1)前台启动Presto，控制台显示日志
bin/launcher run

(2)后台启动Presto
bin/launcher start
```

#### <a name="6">2.2Presto命令行Client安装


1）下载Presto客户端

https://repo1.maven.org/maven2/com/facebook/presto/presto-cli/0.208/presto-cli-0.208-executable.jar

2）修改名称为prestocli，将文件移到presto目录下，并为其增加执行权限

3）启动prestocli

```
./prestocli --server hadoop102:8881 --catalog hive --schema default
```

4）Presto命令行操作

Presto的命令行操作，相当于Hive命令行操作。每个表必须要加上schema

```
select * from schema.table limit 100
```

#### <a name="7">2.3Presto可视化Client安装


1）上传unzip yanagishima-18.0.zip至/opt/module

```
unzip unzip yanagishima-18.0.zip
```

2）解压后进入conf文件夹,vim yanagishima.properties

```
jetty.port=7080
presto.datasources=yingzi-presto
presto.coordinator.server.yingzi-presto=http://hadoop102:8881
catalog.yingzi-presto=hive
schema.yingzi-presto=default
sql.query.engines=presto
```

3）在/opt/module/yanagishima-18.0路径下启动yanagishima

```
nohup bin/yanagishima-start.sh >y.log 2>&1 &
```

4）启动web页面

http://hadoop102:7080

![](https://yingziimage.oss-cn-beijing.aliyuncs.com/img/202302151556108.png)


### <a name="8">3.Presto优化之数据存储


#### <a name="9">3.1合理设置分区


​		与Hive类似，Presto会根据元数据信息读取分区数据，合理的分区能减少Presto数据读取量，提升查询性能

#### <a name="10">3.2使用列式存数


​		Presto对ORC文件读取做了特定优化，因此在Hive中创建Presto使用的表时，尽量使用ORC格式存储

#### <a name="11">3.3使用压缩


​		数据压缩可以减少节点见数据传输对IO带宽压力，对于即席查询需要快速解压，可采用Snappy压缩

### <a name="12">4.Persto优化之查询SQL


#### <a name="13">4.1只选择使用的字段


​		由于采用列式存储，选择需要的字段可加快字段的读取、减少数据量。避免采用*读取所有字段。

#### <a name="14">4.2过滤条件必须加上分区字段


​		对于有分区的表，where语句中优先使用分区字段进行过滤。

#### <a name="15">4.3Group By语句优化


​		合理安排Group by 语句中字段顺序对性能有一定提升。

#### <a name="16">4.4Order by使用Limit


​		Order by需要扫码数据到单个worker节点进行排序，导致单个worker需要大量内存。如果是查询Top N或者Bottom N,使用Limit可减少排序计算和内存压力

#### <a name="17">4.5使用Join语句时将大表放在左边


​		Presto中join的默认算法是broadcast join,即将join左边的表分割到多个worker，然后将join右边的表数据整个复制一份发生到每个worker进行计算。如果右边的表数据量太大，则可能会报内存溢出错误。

### <a name="18">5.注意事项


#### <a name="19">5.1字段名引用


​		避免和关键字冲突：MySQL对字段加反引号、Presto对字段加双引号分割。

​		当然，如果字段名称不是关键字，可以不加这个双引号

#### <a name="20">5.2时间函数


​		对于Timestamp，需要进行比较的时候，需要添加Timestamp关键字，而MySQL中对Timestamp可以直接进行比较。

```
/*MySQL的写法*/
SELECT t FROM a WHERE t > '2017-01-01 00:00:00'; 

/*Presto中的写法*/
SELECT t FROM a WHERE t > timestamp '2017-01-01 00:00:00';
```

#### <a name="21">5.3不支持INSERT OVERWRITE语法


​		Presto中不支持insert overwrite语法，只能先delete，然后insert into


#### <a name="22">5.4PARQUET格式


​		Presto目前支持Parquet格式，支持查询，但不支持insert
