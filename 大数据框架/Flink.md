&emsp;<a href="#0">Flink</a>  
&emsp;&emsp;<a href="#1">Flink介绍</a>  
&emsp;&emsp;<a href="#2">Flink架构(重点)</a>  
&emsp;&emsp;<a href="#3">作业提交流程</a>  
&emsp;&emsp;&emsp;<a href="#4">高层级视角</a>  
&emsp;&emsp;&emsp;<a href="#5">独立模式</a>  
&emsp;&emsp;&emsp;<a href="#6">YARN集群</a>  
&emsp;&emsp;<a href="#7">Flink的水位线(重点)</a>  
&emsp;&emsp;<a href="#8">Flink的窗口(重点)</a>  
&emsp;&emsp;&emsp;<a href="#9">窗口分类</a>  
&emsp;&emsp;&emsp;<a href="#10">窗口函数</a>  
&emsp;&emsp;&emsp;<a href="#11">窗口其他API</a>  
&emsp;&emsp;<a href="#12">Flink的Checkpoint(重点)</a>  
&emsp;&emsp;&emsp;<a href="#13">checkpoint保存</a>  
&emsp;&emsp;&emsp;<a href="#14">checkpoint恢复</a>  
&emsp;&emsp;&emsp;<a href="#15">checkpoint算法</a>  
&emsp;&emsp;&emsp;<a href="#16">checkpoint配置</a>  
&emsp;&emsp;&emsp;<a href="#17">Savepoint</a>  
&emsp;&emsp;<a href="#18">Exactly-One(重点)</a>  
&emsp;&emsp;&emsp;<a href="#19">概念</a>  
&emsp;&emsp;&emsp;<a href="#20">输入端保证</a>  
&emsp;&emsp;&emsp;<a href="#21">输出端保证</a>  
&emsp;&emsp;<a href="#22">Flink的CEP(重点)</a>  
&emsp;&emsp;&emsp;<a href="#23">概念</a>  
&emsp;&emsp;&emsp;<a href="#24">应用场景</a>  
&emsp;&emsp;&emsp;<a href="#25">模式API</a>  
&emsp;&emsp;&emsp;<a href="#26">模式的检测处理</a>  
&emsp;&emsp;<a href="#27">Flink处理背压</a>  
&emsp;&emsp;<a href="#28">Flink SQL解析过程</a>  
## <a name="0">Flink


### <a name="1">Flink介绍


流式大数据处理引擎

- 内存执行速度 -> 速度快
- 任意规模 -> 可扩展性强

Flink区别与传统数据处理框架特性如下：

- 高吞吐、低延迟：每秒处理数百万个事件，毫秒级延迟
- 结果的准确性：提供事件事件、处理时间语义。对于乱序事件流仍然能提供一致且准确的结果
- exactle-once状态一致性保证
- 高可用：本身高可用的设置，加上与K8s、YARN、Mesos的紧密集成，再加上从故障中快速恢复、动态扩展任务的能力，Flink能做到以极少的停机事件 7 * 24 全体候运行
- 能够更新应用程序代码将作业迁移到不同的Flink集群，而不会丢失应用程序状态

<img src="https://yingziimage.oss-cn-beijing.aliyuncs.com/img/202206212301500.png" style="zoom:67%;" />

### <a name="2">Flink架构(重点)


<img src="https://yingziimage.oss-cn-beijing.aliyuncs.com/img/image-20220430170918410.png" style="zoom:67%;" />

架构组成：**作业管理器**（JobManager），**任务管理器**（TaskManagers）

启动方式：

- 作为独立集群的进程，直接在机器上启动
- 在容器中启动
- 由资源管理平台调度启动，如YARN、K8S

<u>**JobManager**</u>

Flink集群中任务管理和调度的核心，是控制应用执行的主进程

包含三个组件：JobMastser、ResourceManager、Dispatcher

**1、JobMaster**

```
是JobManager最核心的组件，负责所有需要中央协调的操作，也负责处理单独的Job(一一对应)
在Job提交时，JobMaster会先接收要执行的应用(应用一般指客户端提交来的Jar包、数据流图、作业图)
JobMaster会把JobGraph转化成一个物理层面的数据流图，包含了所有可并发执行的任务
JobMaster会向ResourceManager发出请求，申请执行任务必要的资源，当获取足够的资源后就将数据流图发往TaskManager
```

**2、ResourceManager**

```
负责资源的分配和管理，资源指TaskManager的任务槽(task slots).任务槽是集群中资源调配单元，执行计算的一组CPU和内存资源，每一个Task都需要分配到一个slot上执行

当有新的作业申请资源，RM会将空闲槽位的TaskManager分配给JobMaster。若RM没有足够的任务槽，它还可以向资源提供平台发起会话，请求提供启动TaskManager进程的容器。此外RM还负责停掉空闲的TaskManager，释放计算资源

Flink内置的RM和其他资源管理管理平台(如YARN)的RM不同，针对不同的环境和资源管理管理平台(Standalone、YARN)有不同的具体实现
	-Standalone部署时，TaskManager单独启动，没有Per-Job模式，RM只能分发TaskManager的任务槽,不能单独启动新的TaskManager
```

**3、Dispatcher**

```
提供一个REST接口，用来提交应用
为每一个新提交的作业启动一个新的JobMaster组件
启动Web UI,展示、监控作业执行的信息
```

<u>**TaskManager**</u>

Flink中的工作进程，数据流的具体计算者，被称为Worker

```
Flink集群中至少有一个TaskManager，分布式计算会有多个
每个TaskManager都包含一定数量的任务槽(task slots)，slot的数量现在TaskManager能够并行处理的任务数量

启动之后，TaskManager会向RM注册它的slots，收到RM指令后，TaskManager会将一个或者多个槽位提供给JobMaster调用，JobMaster分配任务来执行
在执行过程中，TaskManager可以缓存数据，还可以和运行于同一应用的其他TaskManager交换数据
```

### <a name="3">作业提交流程


#### <a name="4">高层级视角


![](https://yingziimage.oss-cn-beijing.aliyuncs.com/img/image-20220430182738138.png)

```
1、客户端(APP)通过分发器提供的REST接口，将作业提交给JobManager
2、由分发器启动JobMaster，并将作业(含JobGraph)提交给JobMaster
3、JobMaster将JobGraph解析为可执行的ExxecutionGraph，得到所需的资源数量，然后向RM请求资源(slots)
4、RM判断当前是否有足够的可用资源，如果没有就启动新的TaskManager
5、TaskManager启动之后，向RM注册自己的可用任务槽
6、RM通知TaskManager为新的作业提交slots
7、TaskManager连接到对应的JobMaster，提供slots
8、JobMaster将需要执行的任务分发给TaskManager
9、TaskManager执行任务，互相之间可以交换数据
```

#### <a name="5">独立模式


<img src="https://yingziimage.oss-cn-beijing.aliyuncs.com/img/image-20220430183110363.png" style="zoom:67%;" />

```
在独立模式下，只有会话模式和应用模式，两者整体来看流程十分相似：TaskManager都需要手动启动，当RM收到JobMaster的请求时，会要求TaskManager提供资源。

JobMaster启动时间点，会话模式预先启动，应用模式则在作业提交时启动
```

#### <a name="6">YARN集群


有三类模式：**会话模式**(Session)、**单作业模式**(Per-Job)、**应用模式**(Application)

**1、会话模式**

<img src="https://yingziimage.oss-cn-beijing.aliyuncs.com/img/image-20220430184407126.png" style="zoom:67%;" />

<img src="https://yingziimage.oss-cn-beijing.aliyuncs.com/img/image-20220430184424329.png" style="zoom:67%;" />

```
在会话模式下，需要先启动一个YARN session。这个会话会创建一个Flink集群，集群只启动JobManager，而TaskManager根据需要动态启动。在JobManager内部，由于还没有提交作业，只有RM、Dispatcher在运行

1、客户端通过REST接口，将作业提交给分发器
2、分发器启动JobMaster，并将作业(含JobGraph)提交给JobMaster
3、JobMaster向资源管理器请求资源（slots）
4、资源管理器向YARN的资源管理器请求container资源
5、YARN启动新的TaskManager容器
6、TaskManager启动之后，Flink的资源管理器注册自己的可用任务槽
7、资源管理器通知TaskManager为新的作业提供slots
8、TaskManager连接到对应的JobMaster，提供slots
9、JobMaster将需要执行的任务分发给TaskManager，执行任务
```

**2、单作业**

<img src="https://yingziimage.oss-cn-beijing.aliyuncs.com/img/image-20220430190328328.png" style="zoom:67%;" />

```
1、客户端将作业提交给YARN的RM，同时将Flink的Jar包和配置上传到HDFS，以便后续启动Flink相关组件的容器
2、YARN的RM分配Container资源，启动Flink JobManager，并将作业提交给JobMaster，这里省略了Dispatcher组件
3、JobMaster向RM请求资源(slots)
4、RM向YARN的RM请求container资源
5、YARN启动新的TaskManager容器
6、TaskManager启动之后，向Flink的RM注册自己的可用任务槽
7、RM通知TaskManager为新的作业提供slots
8、TaskManager连接到对应的JobMaster，提供slots
9、JobMaster将需要执行的任务分发给TaskManager，执行任务
```

**3、应用模式**

```
与作业模式的提交流程非常相似，只是初始提交给YARN的RM不再是具体作业，而是整个应用。每个应用可能包含多个作业
```

### <a name="7">Flink的水位线(重点)


先引申Flink中的时间语义：**处理时间**、**事件时间**

- 处理时间：执行处理操作的机器的的系统时间
- 事件时间：数据生成的时间，自带一个时间戳(timestamp)

**水位线定义**：在事件时间的语义下，不依赖于系统时间，基于数据自带的时间戳去定义一个时钟，表示当前时间的进展

**水位线的特性**：

- 基于数据的时间戳生成
- 插入数据流中的一个标记
- 通过设置延迟保证正确处理乱序数据
- 水位线t，代表数据流中事件时间已经到达t，时间戳小<= t的数据均到齐了

**水位线的生成策略**：DataStream API中单独生成水位线的方法：assignTimestampsAndWatermarks()，传入watermarkStrategy参数

```java
 public SingleOutputStreamOperator<T> assignTimestampsAndWatermarks(
         WatermarkStrategy<T> watermarkStrategy)

public interface WatermarkStrategy<T> extends TimestampAssignerSupplier<T>,
	WatermarkGeneratorSupplier<T>{
 		@Override
 		TimestampAssigner<T> createTimestampAssigner(TimestampAssignerSupplier.Context context);
 		@Override
 		WatermarkGenerator<T> createWatermarkGenerator(WatermarkGeneratorSupplier.Context context);
}
```

- TimestampAssigner：负责从流中数据元素的某个字段提取时间戳，并分配给元素

- WatermarkGenerator：按照既定方式，基于时间戳生成水位线。接口有两个方法：onEvent，onPeriodcEmit

  - onEvent：每个数据到来时都会调用的方法，它的参数有当前数据、时间戳、WatermarkOutput

  - onPeriodcEmit：周期性调用的方法，由WatermarkOutput发出水位线。周期时间为处理时间，可调用环境配置的setAutoWatermarkInterva()方法来设置，默认为200ms

    ```java
    public interface WatermarkGenerator<T> {
        void onEvent(T event, long eventTimestamp, WatermarkOutput output);
        void onPeriodicEmit(WatermarkOutput output);
    }
    ```

**内置水位线生成器**：有序 -> forMonotonousTimestamps()、无序 -> forBoundedOutOfOrderness()

```java
这两个等价：
WatermarkStrategy.forMonotonousTimestamps()
WatermarkStrategy.forBoundedOutOfOrderness(Duration.ofSeconds(0))
```

注意：乱序流中生成的水位线真正的时间戳：最大时间戳 - 延迟时间 - 1，单位为毫秒

```java
public void onPeriodicEmit(WatermarkOutput output) {
 	output.emitWatermark(new Watermark(maxTimestamp - outOfOrdernessMillis - 1));
}
```

**水位线的传递**：实际应用中上下游都有多个并行子任务，为了统一推进事件时间的进展，要求上游任务处理完水位线、时钟改变之后，将当前水位线广播给所有下游任务。上游并行子任务发来不同的水位线，当前任务会为每一个分区设置一个分区水位线，当前任务时钟取最小水位线

<img src="https://yingziimage.oss-cn-beijing.aliyuncs.com/img/image-20220430202734825.png" style="zoom: 67%;" />

**水位线的总结**：

- 在事件时间的世界里面，承担了时钟的角色，是唯一的时间尺度
- 水位线的默认计算公式：水位线 = 观察到的最大事件时间 - 最大延迟时间 - 1 ms
- 数据流开始时，插入一个负无穷大的水位线；水位线结束时，插入一个正无穷大的水位线，以保证所有窗口合闭、所有定时器被触发
- 对于离线数据集，Flink将其作为流读入。Flink只会插入两次水位线，最开始插入负无穷大，结束处插入正无穷大



### <a name="8">Flink的窗口(重点)


将无限数据切割成有限大数据块，处理无界流的核心

<img src="https://yingziimage.oss-cn-beijing.aliyuncs.com/img/image-20220430203808594.png" alt="image-20220430203808594" style="zoom: 50%;" />

窗口如桶，将每个数据发到对应的桶中，当到达窗口结束时间时，就对每个桶中收集的数据进行计算处理

- 动态创建：当有落在这个窗口区间范围的数据到达时，才会创建对应的窗口
- 窗口关闭：到达窗口结束时间时，窗口就触发计算并关闭

#### <a name="9">窗口分类


**<u>按照驱动型分类</u>**：时间窗口、计数窗口

```java
1、时间窗口
窗口大小 = 结束时间 - 开始时间
Flink有一个专门的TimeWindow类表示时间窗口，该类只有两个私有属性：
    private final long start;
	private final long end;
可以调用公有getStart()和getEnd()方法直接获取这两个时间戳。另外提供了maxTimeStamp()获取窗口中包含的最大时间戳
public long maxTimestamp(){
    return end - 1;
}
左闭右开[start,end)

2、计数窗口
基于元素的个数来截取数据，到达固定个数时就触发计算并关闭窗口
底层通过全局窗口(Global Window)实现
```

**<u>按照窗口分配数据的规则分类</u>**：滚动窗口(Tumbling)、滑动窗口(Sliding)、会话窗口(Session)、全局窗口(Global)

**滚动窗口**

<img src="https://yingziimage.oss-cn-beijing.aliyuncs.com/img/image-20220430205445330.png" style="zoom: 50%;" />

```
滚动窗口有固定的大小，是一种对数据进行均匀切片的划分方式，首尾相接，每个数据都会被分配且只属于一个窗口
滚动窗口可以基于时间、个数定义
```

**滑动窗口**

<img src="https://yingziimage.oss-cn-beijing.aliyuncs.com/img/image-20220430205911348.png" style="zoom:67%;" />

````
滑动窗口大小固定，但窗口之间并不是首尾相接，有部分重合
滑动窗口可以基于时间、个数定义
定义滑动窗口的两个参数：窗口大小、滑动步长。数据分配到多个窗口的个数 = 窗口大小 / 滑动步长
````

**会话窗口**

<img src="https://yingziimage.oss-cn-beijing.aliyuncs.com/img/image-20220430210201919.png" style="zoom: 67%;" />

```
会话窗口长度不固定、起始和结束时间不确定，各个分区窗口之间无关联，会话窗口之间不会重叠
会话窗口只能基于时间定义，终止的标志是隔一段时间没有数据到来
size：两个会话窗口之间的最小距离。可设置静态固定的size，也可以自定义提取器动态提取最小间隔gap的值
在Flink底层，对会话窗口有特殊的处理：每来一个新数据，都会创建一个新会话窗口，判断已有窗口之间的距离。如果小于给定的size，就将它们合并
```

**全局窗口**

<img src="https://yingziimage.oss-cn-beijing.aliyuncs.com/img/image-20220430210806692.png" style="zoom:67%;" />

```
无界流的数据永无止境，窗口没有结束的时候，默认不做触发计算，若希望对数据进行计算处理，需自定义触发器(Trigger)
相同key的所有数据都会分配到同一个窗口中
```

#### <a name="10">窗口函数


定义窗口分配，决定数据属于哪个窗口；定义窗口函数，如何进行计算

<img src="https://yingziimage.oss-cn-beijing.aliyuncs.com/img/image-20220430211543068.png" style="zoom:67%;" />

由处理方式分为两类：增量聚合函数、全窗口函数

**<u>增量聚合函数</u>**

窗口对无限流的切分，可看作得到一个有界数据集。对每来一条数据立即计算，中间保持一个简单的聚合状态，等到窗口结束时拿出之前聚合的状态直接输出

典型的两个增量函数：归约函数(ReduceFunction)、聚合函数(AggregateFunction)

- **归约函数**：将窗口收集到的数据两两进行归约，实现增量式大聚合

  ```java
  public interface ReduceFunction<T> extends Function, Serializable {
  	T reduce(T value1, T value2) throws Exception;//注意：输入数据类型、聚合状态类型、输出类型一致
  }
  ```

- **聚合函数**：更加灵活的进行窗口聚合操作,AggregateFunction可看作是ReduceFunction的通用版本。输入类型(IN)、累加器类型(ACC)、输出类型(OUT)

  - createAccumulator：创建累加器，为聚合创建初始状态
  - add：将输入元素添加到累加器中，返回一个新的累加器值
  - getResult：从累加器中提取聚合输出的结果
  - merge：合并两个累加器，并将合并的状态作为累加器返回。该方法只在合并窗口场景下调用，一般是会话窗口

  ```java
  public interface AggregateFunction<IN, ACC, OUT> extends Function, Serializable{
      ACC createAccumulator();
      ACC add(IN value, ACC accumulator);
      OUT getResult(ACC accumulator);
      ACC merge(ACC a, ACC b);
  }
  ```

**<u>全窗口函数</u>**

全窗口需要先收集窗口中的数据，并在内部缓存起来，等到窗口要输出结果的时候再取出数据进行计算

典型的批处理思路——攒数据，等一批到齐后再正式启动处理流程

全窗口函数有两种：窗口函数(WindowFunction)、处理窗口(ProcessWindowFunctio)

- **窗口函数**：当窗口到达结束时间时需要触发计算，调用apply()。从input集合中取出窗口收集的数据，结合key和windowx信息，通过收集器输出结果。目前WindowFunction的作用已被ProcessWindowFunction全覆盖

  ```java
  public interface WindowFunction<IN, OUT, KEY, W extends Window> extends Function,Serializable {
      void apply(KEY key, W window, Iterable<IN> input, Collector<OUT> out) throws Exception;
  }
  ```

- **处理窗口函数**：Window API中最底层的通用窗口函数接口，可以获取到一个上下文对象

  ```java
  public abstract class ProcessWindowFunction<IN, OUT, KEY, W extends Window>
          extends AbstractRichFunction {
          public abstract void process(
              KEY key, Context context, Iterable<IN> elements, Collector<OUT> out) throws Exception;
      
          public void clear(Context context) throws Exception {}
      
          public abstract class Context implements java.io.Serializable {
          public abstract W window();
          public abstract long currentProcessingTime();
          public abstract long currentWatermark();
          public abstract KeyedStateStore windowState();
          public abstract KeyedStateStore globalState();
          public abstract <X> void output(OutputTag<X> outputTag, X value);
      }
  }
  ```

**增量聚合和全窗口的总结**

```
增量聚合的优点：高效、输出更加实时
全窗口的优点：提供更多的信息，更加通用的窗口操作

实际应用中可以兼具这两者的优点，结合使用：
第一个参数(增量聚合函数)处理窗口数据，每来一个数据做一次聚合
第二个参数(全窗口函数)等到窗口需要触发计算时，进行逻辑输出，此时全窗口函数不再缓存所有数据，而将增量聚合的几个当做Iterable输出
```

#### <a name="11">窗口其他API


对于窗口算子，窗口分配器和窗口函数是必须的。选用其他一些API能更加灵活控制窗口行为：**触发器**、**移除器**、**允许延迟**、**侧输出流**

**触发器**

Trigger是窗口算子的内部属性，每个窗口分配器都会有一个对应的默认触发器，Flink内置窗口类型的触发器都已经实现

所有事件时间窗口，默认的触发器都是EventTimeTrigger；类似还有ProcessingTimeTrigger、CountTrigger

```java
public abstract class Trigger<T, W extends Window> implements Serializable {
    public abstract TriggerResult onElement(T element,long timestamp,W window,TriggerContext ctx) throws Exception;
    public abstract TriggerResult onProcessingTime(long time, W window, TriggerContext ctx) throws Exception;
    public abstract TriggerResult onEventTime(long time, W window, TriggerContext ctx) throws Exception;
    public abstract void clear(W window, TriggerContext ctx) throws Exception;
    
    public boolean canMerge() {return false;}
    public void onMerge(W window, OnMergeContext ctx) throws Exception {
        throw new UnsupportedOperationException("This trigger does not support merging.");
    }
    
    public interface TriggerContext {
        long getCurrentProcessingTime();
        MetricGroup getMetricGroup();
        long getCurrentWatermark();
        void registerProcessingTimeTimer(long time);
        void registerEventTimeTimer(long time);
        void deleteProcessingTimeTimer(long time);
        void deleteEventTimeTimer(long time);
        <S extends State> S getPartitionedState(StateDescriptor<S, ?> stateDescriptor);
        @Deprecated
        <S extends Serializable> ValueState<S> getKeyValueState( String name, Class<S> stateType, S defaultState);
        @Deprecated
        <S extends Serializable> ValueState<S> getKeyValueState(String name, TypeInformation<S> stateType, S defaultState);
    }
        public interface OnMergeContext extends TriggerContext {
        <S extends MergingState<?, ?>> void mergePartitionedState(
                StateDescriptor<S, ?> stateDescriptor);
    }
    
```

- onElement：窗口中每来一个元素，调用该方法
- onEventTime：注册的事件时间定时触发时，调用该方法
- onProcessingTime：注册的处理时间定时器触发时，调用该方法
- clear：窗口关闭销毁时，调用该方法，用来清除自定义的状态

上面的前三个方法返回类型都是TriggerResult，这是一个枚举类型，其中定义了对窗口进行操作的四种类型

- CONTINUE(继续)：什么都不做
- FIRE(触发)：触发计算，输出结果
- PURGE(清除)：清空窗口中所有数据，销毁窗口
- FIRE_AND_PURGE(触发并清除)：触发计算输出结果，并清除窗口

**移除器**

用来定义移除某些数据的逻辑，默认情况下，预实现的移除器都是在执行窗口函数之前移除数据的

```java
public interface Evictor<T, W extends Window> extends Serializable {
    void evictBefore(
            Iterable<TimestampedValue<T>> elements,
            int size,
            W window,
            EvictorContext evictorContext);
    
        void evictAfter(
            Iterable<TimestampedValue<T>> elements,
            int size,
            W window,
            EvictorContext evictorContext);
    
    interface EvictorContext {
        long getCurrentProcessingTime();
        MetricGroup getMetricGroup();
        long getCurrentWatermark();
    }
```

- evictBefore：执行窗口函数之前的移除数据操作
- evictAfter：执行窗口函数之后的移除数据操作

**允许延迟**

事件时间语义下，乱序流中可能出现数据迟到的情况，水位线并不一定能保证时间戳更早的所有数据不会再来，为窗口设置一个运行的最大延迟。水位线 = 窗口结束时间 + 延迟时间

```java
    public WindowedStream<T, K, W> allowedLateness(Time lateness) {
        builder.allowedLateness(lateness);
        return this;
    }
```

**侧输出流**

将未收入窗口的迟到数据，放入侧输出流(side output)进行另外的处理。

sideOutputLateData() 方法，传入一个输出标签，用来标记的迟到数据流

```java
    public WindowedStream<T, K, W> sideOutputLateData(OutputTag<T> outputTag) {
        outputTag = input.getExecutionEnvironment().clean(outputTag);
        builder.sideOutputLateData(outputTag);
        return this;
    }
```

getSideOutput()方法，传入对应的输出标签，获取迟到数据所在的流

```java
public <X> DataStream<X> getSideOutput(OutputTag<X> sideOutputTag) {
        sideOutputTag = clean(requireNonNull(sideOutputTag));
        sideOutputTag = new OutputTag<X>(sideOutputTag.getId(), sideOutputTag.getTypeInfo());
        TypeInformation<?> type = requestedSideOutputs.get(sideOutputTag);
        if (type != null && !type.equals(sideOutputTag.getTypeInfo())) {
            throw new UnsupportedOperationException(
                    "A side output with a matching id was "
                            + "already requested with a different type. This is not allowed, side output "
                            + "ids need to be unique.");
        }
        requestedSideOutputs.put(sideOutputTag, sideOutputTag.getTypeInfo());
        SideOutputTransformation<X> sideOutputTransformation =
                new SideOutputTransformation<>(this.getTransformation(), sideOutputTag);
        return new DataStream<>(this.getExecutionEnvironment(), sideOutputTransformation);
    }
```



### <a name="12">Flink的Checkpoint(重点)


Flink容错机制的核心。检查是针对故障恢复的结果而言，在有状态的流处理中，任务继续处理新数据，不需要之前的计算结果，而是需要任务之前的状态。故障恢复之后应该于发生故障前完全一致，需要检查结果的正确性，因此Checkpoint又称为一致性检查点

#### <a name="13">checkpoint保存


- 检查点保存：周期性地触发保存

- 保存的时间点：当所有任务都处理完同一条输入数据时，对任务状态做快照保存下来。

- 检查点存储：将快照写入外部存储，由状态后端的检查点存储(CheckpointStorage)来决定，有两种选择：

  作业管理器的堆内存(JobManagerCheckpointStorage)、文件系统(FileSystemCheckpointStorage)

#### <a name="14">checkpoint恢复


在允许流处理程序是，Flink周期性地保存检查点，当发生故障时，找到最近一次成功保存的检查点来恢复状态

<img src="https://yingziimage.oss-cn-beijing.aliyuncs.com/img/image-20220501112234760.png" style="zoom:50%;" />

已处理完三个数据后保存了一个检查点，之后又正常处理一个数据flink，在处理第五个数据hello时发生故障。这里Source任务处理完毕，偏移量为5，Map任务已经完成，在Sum任务处理中发生故障，此时状态并未保存，需要从检查点来恢复状态

**(1)重启应用**

遇到故障后，第一步重启，将所有任务的状态清空

<img src="https://yingziimage.oss-cn-beijing.aliyuncs.com/img/image-20220501112621999.png" style="zoom:50%;" />

**(2)读取检查点，重置状态**

找到最后一次保存的检查点，从中读取每个算子任务状态的快照，填充到对应的状态中。此时恢复到第三个数据处理完毕，key为flink的数据还没有来

<img src="https://yingziimage.oss-cn-beijing.aliyuncs.com/img/image-20220501112740914.png" style="zoom:50%;" />

**(3)重放数据**

从保存检查点后开始重新读取数据，通过Source任务向外部数据源重新提交偏移量

<img src="https://yingziimage.oss-cn-beijing.aliyuncs.com/img/image-20220501112902508.png" style="zoom: 67%;" />

**(4)继续处理数据**

接下来正常处理数据即可，当处理到5个数据时，就已经追上发生故障时的系统状态了

<img src="https://yingziimage.oss-cn-beijing.aliyuncs.com/img/image-20220501113101896.png" style="zoom: 67%;" />

#### <a name="15">checkpoint算法


**检查点分界线(Barrier)**：借鉴水位线设计，在数据流中插入一个特殊的数据结构，专门用来表示触发检查点保存的时间点

- JobManager有一个检查点协调器(checkpointcoordinator),专门用来协调处理检查点的相关工作，定期向JobManager发出指令，要求保存检查点
- TaskManager会让所有Source任务把偏移量保存起来，并将带有检查点ID的分界线插入到当前数据流
- 每个算子任务只要处理到该分界线，就会将当前状态进行快照

<img src="https://yingziimage.oss-cn-beijing.aliyuncs.com/img/image-20220501114239120.png" style="zoom: 67%;" />

**分布式快照算法**：Flink使用了Chandy-Lamport算法的一种变体，被称为异步分界线快照算法(asynchronous barrier snapshotting)

- 当上游任务向多个并行下游任务发送barrier时，需要广播出去
- 当多个上游任务向同一个下游任务发生barrier时，需要在下游任务执行分界线对齐操作，等所有并行分区的barrier都到齐，才开始状态保存

为了详细介绍检查点算法原理，以下对WordCount程序进行扩展，考虑所有算子并行度为2的场景

<img src="https://yingziimage.oss-cn-beijing.aliyuncs.com/img/image-20220501115923385.png" style="zoom:50%;" />

检查点算法保存的具体过程如下：

**(1)JobManager发送指令，触发检查点的保存；在Source任务中插入分界线，将偏移量保存**

JobManager会周期性地向每个TaskManager发送一条带有新检查点ID的消息，通过这种方式启动检查点，收到指令后，TaskManager会在所有Source任务中插入分界线，并将偏移量保存到远程的持久化存储中

<img src="https://yingziimage.oss-cn-beijing.aliyuncs.com/img/image-20220501120036749.png" style="zoom: 67%;" />

**(2)状态快照保存，分界线向下游传递**

偏移量存入持久化存储后，会返回通知Sourcer任务，Source任务就会向JobManager确认检查点完成，随后将barrier向下游传递

<img src="https://yingziimage.oss-cn-beijing.aliyuncs.com/img/image-20220501120427595.png" style="zoom:50%;" />

**(3)向下游多个并行子任务广播分界线，执行分界线对齐**

Map任务没有状态，故直接将barrier继续向下游传递，这时由于进行keyBy分区，故需要将barrier广播到下游并行的两个Sum任务。同时，Sum任务可能收到来自上游两个并行Map任务的barrier，故需要执行分界线对齐

<img src="https://yingziimage.oss-cn-beijing.aliyuncs.com/img/image-20220501120755820.png" style="zoom:50%;" />

**(4)分界线对齐后，保存状态到持久化存储**

各个分区的分界线都对齐后，就可以对当前状态做快照，保存到持久化存储，存储完成通知JobManager保存完毕，将barrier继续向下游传递。整个过程中，每个任务保存自己的状态都是相对独立的，互不影响

<img src="https://yingziimage.oss-cn-beijing.aliyuncs.com/img/image-20220501121123600.png" style="zoom:50%;" />

**(5)先处理缓存数据，然后正常继续处理**

分界线对齐要求先达到的分区做缓存等待，一定程度上会影响处理速度，当出现背压(backpressure)时，下游任务会堆积大量的缓冲数据，检查点可能需要很久才可以保存完毕。Flink1.11之后提供了不对齐的检查点保存方式，可将未处理的缓冲数据也保存进检查点，此时当我们遇到一个分区barrier时就不需要等待对齐了，而是直接启动状态的保存

当JobManager收到所有任务成功保存状态的信息，就可以确认当前检查点成功保存，之后遇到故障就可以从这里恢复了

#### <a name="16">checkpoint配置


检查点的作用是为了故障恢复，我们不能因为保存检查点占据大量时间，导致数据处理性能明显降低。可在代码中对检查点进行配置

**1、启动检查点**

默认情况下，Flink程序是禁用检查点，启动保存快照功能

```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
env.enableCheckpointing(1000);// 每隔 1 秒启动一次检查点保存，默认500ms
```

对性能影响小，调大间隔时间；故障重启后迅速赶上实时地数据处理，调小间隔时间

**2、检查点存储**

检查点持久化存储位置，取决于CheckpointStorage的设置，默认存储在JobManager的堆内存中

两种方案：作业管理器的堆内存(JobManagerCheckpointStorage)、文件系统(FileSystemCheckpointStorage)

```java
// 配置存储检查点到 JobManager 堆内存
env.getCheckpointConfig().setCheckpointStorage(new JobManagerCheckpointStorage());
// 配置存储检查点到文件系统
env.getCheckpointConfig().setCheckpointStorage(new FileSystemCheckpointStorage("hdfs://namenode:40010/flink/checkpoints"));
```

**3、其他高级配置**

通过检查点配置来进行设置：

```java
CheckpointConfig checkpointConfig = env.getCheckpointConfig();
```

- **检查点模式**(CheckpointingMode)：设置检查点一致性的级别，精确一次(exactlt-once)和至少一次(at-least-once)两项，默认精确一次
- **超时时间**(checkpointTimeout)：指定检查点保存的超时时间，超时没完成就会被丢弃掉
- **最小间隔时间**(minPauseBetweenCheckpoints)：指定上一个检查点完成之后，检查点协调器最快等多久可以保存下一个检查点的指令。指定该参数时，maxConcurrentCheckpoints强制设为1
- **最大并发检查点数量**(maxConcurrentCheckpoints)：指定运行中的检查点最多可以有多少个
- **开启外部持久化存储**(enableExternalizedCheckpoints)：默认在作业失败时不会自动清理，释放空间需要手动清理
  - DELETE_ON_CANCELLATION：作业取消时会自动删除外部检查点，若作业失败退出会保留检查点
  - RETAIN_ON_CANECLLATION：作业取消时保留外部检查点
- **检查点异常任务失败**(failOnCheckpointingErrors)：指定检查点发生异常时，是否应该让任务直接失败退出，默认为true
- **不对齐检查点**(enableUnalignedCheckpoints)：不再执行检查点的分界线对齐操作，启用之后可以大大减少产生背压时的检查点保存时间，设置要求检查点模式为exactly-once，并且并非检查点个数为1

```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
// 启用检查点，间隔时间 1 秒
env.enableCheckpointing(1000);
CheckpointConfig checkpointConfig = env.getCheckpointConfig();
// 设置精确一次模式
checkpointConfig.setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE);
// 最小间隔时间 500 毫秒
checkpointConfig.setMinPauseBetweenCheckpoints(500);
// 超时时间 1 分钟
checkpointConfig.setCheckpointTimeout(60000);
// 同时只能有一个检查点
checkpointConfig.setMaxConcurrentCheckpoints(1);
// 开启检查点的外部持久化保存，作业取消后依然保留
checkpointConfig.enableExternalizedCheckpoints(
 ExternalizedCheckpointCleanup.RETAIN_ON_CANCELLATION);
// 启用不对齐的检查点保存方式
checkpointConfig.enableUnalignedCheckpoints();
// 设置检查点存储，可以直接传入一个 String，指定文件系统的路径
checkpointConfig.setCheckpointStorage("hdfs://my/checkpoint/dir")
```

#### <a name="17">Savepoint


除了检查点外，Flink提供了另一个非常独特的镜像保存功能——保存点(Savepoint)

它的原理和算法与检查点完全相同，通过检查点机制来创建流式作业状态的一致性镜像(consistent image)，多了额外的元数据

保存点中的状态快照，是以算子ID和状态名称组织的，相当于一个键值对。从保存点启动应用程序时，Flink会将保存点的状态数据重新分配给相应的算子任务

**<u>保存点的用途</u>**

保存点与检查点最大的区别就是触发的时机。

- 检查点：由Flnk自动管理，定期创建，发生故障之后自动读取进行恢复，自动存盘的功能。主要用来做故障恢复，容错机制的核心
- 保存点：由用户明确地手动触发保存，手动存盘。有计划的手动备份和恢复

保存点可以当做一个强大的运维工具使用，在需要的时候创建保存点，然后停止应用，做一些处理调整之后再从保存点重启

- **版本管理和归档存储**：对重要节点手动备份，设置为某一版本，归档存储应用程序的状态

- **更新Flink版本**：当Flink版本升级时，不需要重新执行所有计算，只要创建一个保存点，停掉应用、升级Flink后，从保存点重启处理

- **更新应用程序**：在程序兼容情况下，状态的拓扑结构和数据类型都不变，直接更新应用程序，从之前的保存点加载

  （修复应用应用程序中的逻辑Bug，更新后接着处理；应用于不同业务逻辑的场景，如A/B测试等）

- **调整并行度**：在应用运行的过程中，发现需要的资源不足或已经有了大量剩余，可以通过保存点重启的方式，将应用程序的并行度增大或减少

- **暂停应用程序**：有时我们不需要调整集群或更新程序，只是单纯地希望把应用暂停、释放一些资源来处理更处理更加重要的应用程序，使用保存点就可以灵活实现应用的暂停、重启，可对有限的集群资源做最好的优化配置

注意：保存点能够在程序更改的时候依然兼容，前提是状态的拓扑结构和数据类型不变。保存点中状态都是以算子ID-状态名称这样的key-value组织起来，算子ID可在代码中直接调用SingleOutputStreamOperator的uid()来进行指定

```java
DataStream<String> stream = env
     .addSource(new StatefulSource())
     .uid("source-id")
     .map(new StatefulMapper())
     .uid("mapper-id")
     .print();
```

对于没有设置DI的算子，Flink默认会自动进行设置，在重新启动应用后可能会导致ID不同而无法兼容以前状态，为了方便后续维护，强烈建议在程序中为每一个算子手动指定ID



### <a name="18">Exactly-One(重点)


#### <a name="19">概念


一致性是结果的正确性，对于分布式系统而言，强调的是不同节点中相同数据的副本应该总是一致的。多个节点并行处理不同的任务，要保证计算结构的正确性，必须不漏掉任何一个数据，不重复处理同一个数据，在发生故障、需要恢复状态进行回滚时就需要更多的保障机制了，通过检查点的保存来保证状态恢复后的结果正确

状态一致性的三种级别：最多一次(AT-MOST-ONCE)、至少一次(AT-LEAST-ONCE)、精确一次(EXACTLY-ONCE)

- 最多一次：任务发生故障时，直接重启。既不恢复丢失的状态，也不会重放丢失的数据，每个数据再正常情况下被处理一次
- 至少一次：所有数据都会丢、都能被处理，有些数据可能被重复处理
- 精确一次：最严格的一致性保证，所有数据有且只会被处理一次

完整的流处理应用，包括了数据源、流处理器、外部存储系统三个部分。这个完整应用的一致性，叫做端到端的状态一致性，取决于三个组件中最弱的一环。能否达到at-least-once一致性级别，主要看数据源能否重放数据。而能否达到exactly-once级别，数据源、流处理器内部、外部存储系统都要有相应的保证机制

对于Flink内部来说，检查点机制可以保证故障故障恢复后数据不丢失，且只处理一次。达到exactly-once的一致性语义了

端到端的一致性关键点：输入数据源端、输出外部存储端

#### <a name="20">输入端保证


Flink读取的外部数据源，对于一些数据源来说并不提供数据的缓冲或持久化保存，数据被消费之后就彻底不存在了，故障后通过检查点恢复到之前状态，但保存检查点到发生故障期间的数据不能重发，就会导致数据丢失，只能保证at-most-once的一致性语义

想要在故障恢复后不丢失数据，外部数据源就必须拥有重放数据的能力。常见的做法就是对数据进行持久化保存，并且可以重设数据的读取位置。一个最经典的应用就是Kafka，在Flink的Source任务中将数据读取的偏移量保存为状态，在故障恢复时从检查点中读取处理，对数据源重置偏移量，重新获取数据

数据可重复 + 检查点 -> at-least-once一致性语义的基本要求

#### <a name="21">输出端保证


数据有可能重复写入外部系统，检查点保存之后，继续到来的数据会一一处理，任务的状态也会更新，最终通过Sink任务将计算结果输出到外部系统

状态改变还没有存入下一个检查点中，这时出现故障，数据都会重新来一遍，计算两次

- 对于内部系统来说，重复计算的动作无影响，状态回滚，最终改变只有一次
- 对于外部系统来说，已经写入的结果无法收回，再次执行就会把同一个数据写入两次

为了实现端到端的exactly-once，需要对外部存储系统、sink连接器有额外的要求，两种方法：幂等写入、事务写入

<u>**幂等写入**</u>

一个操作可以重复执行很多次，但只导致一次结果更改。在数据科学领域，对e^x求导不变；在数据处理领域，对HashMap的插入，相同键值对的重复插入不起作用

并没有真正解决数据重复计算、写入的问题。限制在于外部存储系统必须支持这样的幂等写入，例如，Redis中的键值存储，关系型数据库(如MySQL)中满足查询条件的更新操作

**<u>事务写入</u>**

事务的四个基本特性(ACID)：原子性(Atomicity)、一致性(Correspondence)、隔离性(Isolation)、持久性(Durability)

事务写入的基本思想：用一个事务来进行数据向外部系统的写入，这个事务时与检查点绑定在一起的，当Sink任务遇到barrier时，开始保存状态的同时开启一个事务，接下来所有数据都写入到该事务中。待到检查点保存完毕，将事务提交，所有写入的数据就真正可用了，若中间过程出现故障，状态会回退到上一个检查点，当前事务没有用正常关闭也会回滚，写入外部的数据就将被撤销

有两种实现方式：预写日志(WAL)、两阶段提交(2PC)

**预写日志**

- 1、将结果数据作为日志状态保存起来
- 2、进行检查点保存时，将结果一并做持久化存储
- 3、在收到检查点完成的通知时，将所有结果一次性写入外部系统

DataStream API提供了一个模板类GenericWriteAheadSink，用来实现这种事务型的写入方式

注意：预写日志这种一批写入的方式可能会写入失败，在执行写入动作之后， 必须等待发送成功的返回确认消息，在成功写入所有数据后，在内部再次确认相应的检查点才代表着检查点的真正完成，这里需要将确认信息也进行持久化保存，在故障恢复时，只有存在对应的确认信息，才能保证这批数据已经写入，可以恢复到对应的检查点位置

这种再次确认的方式也会有一些缺陷。当检查点已经成功保存、数据也成功地写入到外部系统，但最终保存确认信息时出现了故障，Flink最终还是会认为没有成功写入。故发生故障，不会使用该检查点，而是需要回退到上一个，会导致这一批数据的重复写入

**两阶段提交**

分成两个阶段：先做预提交，等检查点完成之后再正式提交，这种提交放是真正基于事务的，它需要外部系统提供事务支持

- 1、当第一条数据到来，或收到检查点分界线时，Sink任务都会启动一个事务
- 2、接下来收到的所有数据，都会通过这个事务写入外部系统；这时由于事务没有提交，数据尽管写入了外部系统，不可用，处于预提交状态
- 3、当Sink任务收到JobManager发来检查点完成的通知时，正式提交事务，写入的结果就真正可用了

Flink提供了TwoPhaseCommitSinkFunction接口，方便我们自定义两阶段提交对SinkFunction的实现。两阶段提交精巧，同时对外部系统有较高对要求：

- 外部系统必须提供事务支持，或者Sink任务必须能模拟外边系统上的事务
- 在检查点的间隔期间，必须能够开启一个事务并接受数据写入
- 在收到检查点完成的通知前，事务必须是等待提交的状态，在故障恢复的情况下，可能需要一些时间。如果这时候外部系统关闭事务，那么未提交的数据就会丢失
- Sink任务必须能够在进程失败后恢复事务
- 提交事务必须是幂等操作

### <a name="22">Flink的CEP(重点)


#### <a name="23">概念


复杂事件处理(Complex Event Processing，**CEP**)：针对流处理而言，分析的是低延迟、频繁产生的事件流，检测特定的数据组合

CEP的流出可分为三个步骤：

- 1、定义一个匹配规则
- 2、将匹配规则应用到事件流上，检测满足规则的复杂事件
- 3、对检测到的复杂事件进行处理，得到结果进行输出

模式(**Pattern**)：CEP第一步定义对匹配规则

- 每个简单事件的特征
- 简单事件之间对组合关系

近一步可扩展模式的功能：匹配检测的时间限制；每个简单事件是否可重复出现等

事件的组合关系，称之为近邻关系

- 严格的近邻关系：两个事件之间不能有任何其他事件
- 宽松的近邻关系：相对顺序正确，中介可以有其他事件

可以反向定义，谁后面不能跟谁。模式还可以设定在时间范围内没有满足匹配条件，会导致模式匹配超时

#### <a name="24">应用场景


CEP主要用于实时流数据对分析处理，旨在分析复杂的、看似不相关的事件流中找出有意义的事件组合，进而可以实时地分析判断、输出通知信息或报警

- 风险控制：针对用户的异常行为进行实时检。例如，当用户短时间内频繁登录失败、大量下单却不支付
- 用户画像：对用户行为轨迹进行实时跟踪，从而检测出具有特定行为习惯对一些用户，做出相应的用户画像，精准营销
- 运维监控：对于企业服务对运维管理，利用CEP灵活配置多指标、多依赖来实现更复杂对监控模式

很多大数据框架，如Spark、Samza、Beam等都提供了不同对CEP解决方案，但没有专门的库。Flink提供了，目前CEP的最佳解决方案



#### <a name="25">模式API


Flink CEP的核心是复杂事件的模式匹配

**个体模式**：每个简单事件都有一定的条件规则。个体模式都是以连接词开始定义，比如begin、next等，这些都是Pattern对象对象对方法，返回Pattern对象；个体模式需要一个过滤条件，用来指定具体的匹配规则，一般调用where()实现，传入SimpleCondition内对filter方法

**量词**：给个体模式增加一个量词可以让其循环匹配，接收多个事件。

- oneOrMore()：匹配事件出现至少一次。a.oneOrMore()表示匹配至少1个a的事件组合
- times(times)：匹配事件发生特定次数。a.times(3)表示aaa事件
- times(fromTimes，toTimes)：匹配事件出现的次数范围
- greedy()：只能用在循环模式，使当前循环模式变得贪心。尽可能多的去匹配
- optional()：当前模式可选

```java
// 匹配事件出现 4 次
pattern.times(4);
// 匹配事件出现 4 次，或者不出现
pattern.times(4).optional();
// 匹配事件出现 2, 3 或者 4 次
pattern.times(2, 4);
// 匹配事件出现 2, 3 或者 4 次，并且尽可能多地匹配
pattern.times(2, 4).greedy();
// 匹配事件出现 2, 3, 4 次，或者不出现
pattern.times(2, 4).optional();
// 匹配事件出现 2, 3, 4 次，或者不出现；并且尽可能多地匹配
pattern.times(2, 4).optional().greedy();
// 匹配事件出现 1 次或多次
pattern.oneOrMore();
// 匹配事件出现 1 次或多次，并且尽可能多地匹配
pattern.oneOrMore().greedy();
// 匹配事件出现 1 次或多次，或者不出现
pattern.oneOrMore().optional();
// 匹配事件出现 1 次或多次，或者不出现；并且尽可能多地匹配
pattern.oneOrMore().optional().greedy();
// 匹配事件出现 2 次或多次
pattern.timesOrMore(2);
// 匹配事件出现 2 次或多次，并且尽可能多地匹配
pattern.timesOrMore(2).greedy();
// 匹配事件出现 2 次或多次，或者不出现
pattern.timesOrMore(2).optional()
// 匹配事件出现 2 次或多次，或者不出现；并且尽可能多地匹配
pattern.timesOrMore(2).optional().greedy();
```

**条件**：匹配事件的核心在于匹配条件，选取事件的规则

主要有：简单条件、迭代条件、复合条件、终止条件、调用Pattern对象的subtype()限定匹配事件的子类型

- 简单条件：根据当前事件对特征来决定是否接受它，本质上是filter操作
- 迭代事件：依靠之前事件作判断
- 组合条件：定义多个条件，在外部将其连接。如where().where()
- 终止条件：遇到某个特定事件时当前模型不再循环匹配，调用模式对象的until()。终止条件只有oneOrMore()或者oneOrMore().optional()结合使用
- 限定子类型：调用模式对象的subtype()为当前模式增加子类型限制条件

**模式组**：某些场景需要划分多个阶段，每个阶段又有一连串的匹配规则，我们可以嵌套的方式来定义模式，返回一个GroupPattern，是Pattern的子类

```java
// 以模式序列作为初始模式
Pattern<Event, ?> start = Pattern.begin(
    Pattern.<Event>begin("start_start").where(...)
    	.followedBy("start_middle").where(...)
);
```

**匹配后跳过策略**：用来精准控制循环模式的匹配结果

```java
Pattern.begin("start", AfterMatchSkipStrategy.noSkip())
    .where(...)
```



#### <a name="26">模式的检测处理


将模式应用到事件流上、检测提取匹配的复杂事件，最终得到想要的输出信息

**处理匹配事件**：PatternStream的转换操作主要分为两种，select、process。具体实现是调用API时传入一个函数

**处理超时事件**：在有时间限制的情况下，用within()指定模式检测的时间间隔，超出这个时间那么这组检测应该失败，但又和真正的失败不同，是一种部分成功匹配。在开头能正常匹配的前提下，在规定时间内没等到后续的匹配事件，此时不应该丢弃，需要输出一个提示或报警信息

- **PatternProcessFunction的测输出流**：实现TimedOutPartialMatchHandler接口的processTimedOutMatch()方法，将超时等信息放入一个Map中；在方法内定义一个OutputTag，可通过调用output()方法将超时的部分匹配事件输出到标签所标识的侧输出流中

  ```java
  class MyPatternProcessFunction extends PatternProcessFunction<Event, String>
      implements TimedOutPartialMatchHandler<Event> {
      // 正常匹配事件的处理
      @Override
       public void processMatch(Map<String, List<Event>> match, Context ctx,
      	Collector<String> out) throws Exception{
       	...
       }
   // 超时部分匹配事件的处理
   @Override
   public void processTimedOutMatch(Map<String, List<Event>> match, Context ctx) throws Exception{
       Event startEvent = match.get("start").get(0);
       OutputTag<Event> outputTag = new OutputTag<Event>("time-out"){};
       ctx.output(outputTag, startEvent);
    }
  }
  ```

- **PatternTimeoutFunction**：早期版本用于捕获超时事件的接口，传入三个参数：侧输出标签、超时事件处理函数、匹配事件提取函数

  ```java
  // 定义一个侧输出流标签，用于标识超时侧输出流
  OutputTag<String> timeoutTag = new OutputTag<String>("timeout"){};
  // 将匹配到的，和超时部分匹配的复杂事件提取出来，然后包装成提示信息输出
  SingleOutputStreamOperator<String> resultStream = patternStream 
      .select(timeoutTag,
      // 超时部分匹配事件的处理
       new PatternTimeoutFunction<Event, String>() {
       @Override
   	 public String timeout(Map<String, List<Event>> pattern, long timeoutTimestamp) throws Exception {
           Event event = pattern.get("start").get(0);
           return "超时：" + event.toString();
           }
   	},
  // 正常匹配事件的处理
   new PatternSelectFunction<Event, String>() {
   @Override
   public String select(Map<String, List<Event>> pattern) throws Exception
      {
      ...
       }
   }
  );
  // 将正常匹配和超时部分匹配的处理结果流打印输出
  resultStream.print("matched");
  resultStream.getSideOutput(timeoutTag).print("timeout");
  ```

**处理迟到的数据**：

- Flink CEP沿用设置水位线延迟来处理乱序数据。当第一个事件到来时，先放入一个缓冲区(buffer)，缓冲区内的数据按照时间戳由小到大排序。当一个水位线到来时，将时间戳小于水位线的事件依次取出，进行检测匹配

- 有些事件延迟比较大，以至于水位线早已超过它的时间戳。借鉴窗口做法，基于PatternStream调用sideOutputLateData()方法，将迟到数据放入侧输出流处理

  ```java
  PatternStream<Event> patternStream = CEP.pattern(input, pattern);
  // 定义一个侧输出流的标签
  OutputTag<String> lateDataOutputTag = new OutputTag<String>("late-data"){};
      SingleOutputStreamOperator<ComplexEvent> result = patternStream
       .sideOutputLateData(lateDataOutputTag) // 将迟到数据输出到侧输出流
       .select(
       // 处理正常匹配数据
       new PatternSelectFunction<Event, ComplexEvent>() {...}
       );
  	// 从结果中提取侧输出流
  DataStream<String> lateData = result.getSideOutput(lateDataOutputTag);
  ```

### <a name="27">Flink处理背压


系统在一个临时负载峰值期间接收数据的速率大于其处理速率的一种场景（通俗的讲：接收速度 > 接收速度），处理不当会导致资源耗尽，数据丢失

- 消息发送太快，消息接收太慢，产生消息拥堵
- 发生消息拥堵后，系统会自动降低消息发生速度

**Flink利用自身作为纯数据流引擎的优势来响应背压问题**：Flink运行时主要由operators、streams两大组件构成。每个operator会消费中间态的流，并在流上进行转换，然后生成新的流。Flink使用了高效有界的分布式阻塞队列，一个较慢的接受者会降低发送者的发送效率。队列容量通过缓冲池(LocalBufferPool)来实现，每个生产、消费的流都会被分配以个缓冲区，缓冲被消费后可被回收循环利用

### <a name="28">Flink SQL解析过程


![](https://yingziimage.oss-cn-beijing.aliyuncs.com/img/image-20220502101803380.png)

一条SQL从提交到Calcite解析、优化、执行的步骤如下：

- 1、Sql Parse：sql语句通过java cc解析成语法树(AST)，在calcite中用SqlNode表示AST
- 2、Sql Validator：结合数字字典(catalog)去验证sql语法
- 3、生成Logical Plan：将语法树转换成LoglicalPlan，用relNode表示
- 4、生成optimized LogicalPlan：先基于 calcite rules优化、在基于Flink定制的rules去优化logical Plan
- 5、生成Flink PhysicalPlan：基于Flink里的rules将optimized LogicalPlan转化成Flink的PhysicalPlan
- 6、生产Flink ExecutionPlan：调用相应的tanslatetoPlan方法转换和利用CodeGen元编程成Flink的各种算子

