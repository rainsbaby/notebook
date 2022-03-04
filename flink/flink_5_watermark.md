
## 简介
Watermark的作用？

Watermark 是事件时间域中衡量输入完成进度的一种时间概念。

在处理使用事件时间属性的数据流时，Watermark 是系统测量数据处理进度的一种方法。

假如当前系统的 watermark 为时间 T，那么系统认为所有事件时间小于 T 的消息都已经到达，即系统任务它不会再接收到事件时间小于 T 的消息了。有了 Watermark，系统就可以确定使用事件时间的窗口是否已经完成。

## 核心类

#### DataStream

其中assignTimestampsAndWatermarks方法，表示按指定的WatermarkStrategy生成watermark。

#### WatermarkStrategy

Source中Watermark生成策略。WatermarkGenerator和TimestampAssigner的工厂类。

#### WatermarkGenerator

生成Watermark，可基于event到达或周期性生成。

#### TimestampsAndWatermarksOperator

DataStream. assignTimestampsAndWatermarks方法，得到的对应Transformation为TimestampsAndWatermarksTransformation，对应StreamOperator为TimestampsAndWatermarksOperator，对应关系见StreamGraphGenerator。

TimestampsAndWatermarksOperator负责从event中提取出timestamp和生成watermark。

其中主要包含以下成员，利用TimestampAssigner、WatermarkGenerator即可得到timestamp和watermark，发送给下游。

```
//TimestampsAndWatermarksOperator类
private final WatermarkStrategy<T> watermarkStrategy;

/** The timestamp assigner. */
private transient TimestampAssigner<T> timestampAssigner;

/** The watermark generator, initialized during runtime. */
private transient WatermarkGenerator<T> watermarkGenerator;

/** The watermark output gateway, initialized during runtime. */
private transient WatermarkOutput wmOutput;

/** The interval (in milliseconds) for periodic watermark probes. Initialized during runtime. */
private transient long watermarkInterval;
```

#### 

## 主要流程

### Watermark的生成

Timestamp和Watermark有两种生成方式，一是在source端生成，二是在流中间生成。一般尽量使用第一种方式。

#### StreamRecord中timestamp

```
//StreamRecord类
/** The actual value held by this record. */
private T value;

/** The timestamp of the record. */
private long timestamp;

/** Flag whether the timestamp is actually set. */
private boolean hasTimestamp;
```    

#### Watermark消息

```
//Watermark类
/** The timestamp of the watermark in milliseconds. */
private final long timestamp;
```

#### 1.Timestamp和Watermark在source端生成

接口为SourceFunction.SourceContext。

工厂类为StreamSourceContexts，根据不同的系统时间属性，选择不同的SourceContext。

* 若选择TimeCharacteristic.EventTime，则由ManualWatermarkContext生成Watermark。
* 若选择IngestionTime，则由AutomaticWatermarkContext自动定时生成Watermark，发送给下游。
* 若选择ProcessingTime，则由NonTimestampContext忽略时间戳和watermark。

调用过程为：

StreamSource.run() -> SourceFunction.run() -> SourceContext.collect() / .collectWithTimestamp() / .emitWatermark() 

#### 2.Timestamp和Watermark在流中间生成
使用：DataStream中assignTimestampsAndWatermarks等。生成的Transformation中包含WatermarkStrategy，WatermarkStrategy实现了TimestampAssignerSupplier和WatermarkGeneratorSupplier。

通过 Timestamp Assigners / Watermark Generators 来生成事件时间和 watermark，一般是从消息中提取出时间字段。

调用过程为：

1. Element到达时，TimestampsAndWatermarksOperator.processElement() -> TimestampAssigner.extractTimestamp() -> output.collect() -> WatermarkGenerator.onEvent()

或

2. 定时生成，OperatorChain.createOperator() ->  StreamOperatorFactoryUtil.createOperator() -> StreamTask.getProcessingTimeServiceFactory() ->TimestampsAndWatermarksOperator.onProcessingTime() -> WatermarkGenerator.onPeriodicEmit() -> ProcessingTimeService.registerTimer()触发下一次

#### TimerService
提供定时触发功能，用于定时生成watermark等。


### Watermark的处理

一般来说，operator在向下游转发watermark之前，需要完全处理一个给定的watermark。例如，WindowOperator将首先评估所有应该被触发的窗口，只有在产生由watermark触发的所有输出后，watermark本身才会被发送到下游。换句话说，所有由于发生watermark而产生的元素都将在watermark之前被发射出去。

同样的规则也适用于TwoInputStreamOperator。然而，在这种情况下，运算器的当前watermark被定义为其两个输入的最小值。



## 参考

[Generating Watermarks](https://nightlies.apache.org/flink/flink-docs-release-1.14/docs/dev/datastream/event-time/generating_watermarks/)

