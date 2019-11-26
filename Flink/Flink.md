

## Flink编程模型

input ->  transformation/sink --> output

1. 获取执行环境
2. 获取数据
3. transformation
4. sink
5. 触发执行



## Flink中使用数据源

[Data Sources :](https://ci.apache.org/projects/flink/flink-docs-release-1.8/dev/datastream_api.html#data-sources) Flink comes with a number of pre-implemented source functions, but you can always write your own custom sources by implementing the SourceFunction for non-parallel sources, or by implementing the ParallelSourceFunction interface or extending the RichParallelSourceFunction for parallel sources.

Flink 预实现了很多数据接口，但我们还是可以通过 SourceFunction 自定义数据处理方法，通过实现 SourceFunction 接口实现非并行，通过 ParallelSourceFunction 或 RichParallelSourceFunction 实现并行。

1. 实现 SourceFunction，不能并行处理。
2. 实现 ParallelSourceFunction。
3. 继承 RichParallelSourceFunction。

代码在`course04`



## 自定义Sink总结

1. `RichSinkFunction<T>`,T就是你想要写入对象的类型
2. 重写方法
   1. open/close 生命周期方法
   2. invoke 每条记录执行一次

代码在`course05`



## Flink window

对于Flink里面的三种时间

1. 事件时间  10:30
2. 摄取时间  11:00 
3. 处理时间  11:30

思考：
对于流处理来说，你们觉得应该是以哪个时间作为基准时间来进行业务逻辑的处理呢？

幂等性

env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)
思考：默认的TimeCharacteristic是什么？


窗口分配器：定义如何将数据分配给窗口

A WindowAssigner is responsible for assigning each incoming element to one or more windows
每个传入的数据分配给一个或者多个窗口

1. tumbling windows 滚动窗口
   `have a fixed size and do not overlap`
2. sliding windows  滑动窗口
   `overlapping`
3. session windows  会话窗口
4. global windows   全局窗口

[start timestamp , end timestamp)


https://blog.csdn.net/lmalds/article/details/52704170



## Flink中常用的优化策略

1. 资源
2. 并行度
   	默认是1   适当的调整：好几种  ==> 项目实战
3. 数据倾斜
   	100task  98-99跑完了  1-2很慢   ==> 能跑完 、 跑不完
   	group by： 二次聚合
   		random_key  + random
   		key  - random
   	join on xxx=xxx
   		repartition-repartition strategy  大大
   		broadcast-forward strategy  大小

