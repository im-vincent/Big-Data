### 

Flume介绍

概述

- Flume是一个分布式、可靠、和高可用的海量日志采集、聚合和传输的系统。
- Flume可以采集文件，socket数据包等各种形式源数据，又可以将采集到的数据输出到HDFS、hbase、hive、kafka等众多外部存储系统中一般的采集需求，通过对flume的简单配置即可实现
- Flume针对特殊场景也具备良好的自定义扩展能力，因此，flume可以适用于大部分的日常数据采集场景



运行机制

- Flume分布式系统中最核心的角色是agent，flume采集系统就是由一个个agent所连接起来形成
- 每一个agent相当于一个数据传递员
  [Source 到 Channel 到 Sink之间传递数据的形式是Event事件；Event事件是一个数据流单元。]

```
Event: { headers:{} body: 68 65 6C 6C 6F 0D                               hello. }
Event是Flume数据传输的基本单元
Event = 可选的header + byte array
```



![](http://flume.apache.org/_images/DevGuide_image00.png)

内部有三个组件：

1. Source：采集源，用于跟数据源对接，以获取数据
2. Channel：angent内部的数据传输通道，用于从source将数据传递到sink
3. Sink：下沉地，采集数据的传送目的，用于往下一级agent传递数据或者往最终存储系统传递数据


```shell
# example.conf: A single-node Flume configuration

# a1: agent名称
# r1: source名称
# k1: sink名称
# c1: channel名称

# Name the components on this agent
a1.sources = r1
a1.sinks = k1
a1.channels = c1

# Describe/configure the source
a1.sources.r1.type = netcat
a1.sources.r1.bind = localhost
a1.sources.r1.port = 44444

# Describe the sink
a1.sinks.k1.type = logger

# Use a channel which buffers events in memory
a1.channels.c1.type = memory

# Bind the source and sink to the channel
a1.sources.r1.channels = c1 # a1的的source要发送到那个channel上。
a1.sinks.k1.channel = c1 # a1的sinks要从那个channel接受数据

```





启动方法

```bash
$ bin/flume-ng agent --conf $FLUME_HOME/conf \
--conf-file $FLUME_HOME/conf/example.conf \
--name a1 \
-Dflume.root.logger=INFO,console
```



