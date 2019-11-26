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





-------

TAILDIR

```shell
#-->设置sources名称
a1.sources = sources1
#--> 设置channel名称
a1.channels = fileChannel
#--> 设置sink 名称
a1.sinks = sink1

# source 配置
a1.sources.sources1.type = TAILDIR
a1.sources.sources1.positionFile = /tmp/flume/taildir_position.json
a1.sources.sources1.filegroups = f1
a1.sources.sources1.filegroups.f1 = /u01/app/oracle/diag/rdbms/orcl/orcl/trace/alert_.*.log
a1.sources.sources1.batchSize = 100
a1.sources.sources1.backoffSleepIncrement = 1000
a1.sources.sources1.maxBackoffSleep = 5000
a1.sources.sources1.channels = fileChannel
a1.sources.sources1.headers.f1.headerKey = ora31
a1.sources.sources1.fileHeader = true


# sink1 配置
a1.sinks.sink1.type = file_roll
a1.sinks.sink1.sink.directory = /tmp/flume/flumefiles
a1.sinks.sink1.sink.rollInterval = 0
a1.sinks.sink1.channel = fileChannel

# fileChannel 配置
a1.channels.fileChannel.type = file
#-->检测点文件所存储的目录
a1.channels.fileChannel.checkpointDir = /tmp/flume/checkpoint/oracle_alert/
#-->数据存储所在的目录设置
a1.channels.fileChannel.dataDirs = /tmp/flume/data/oracle_alert/
#-->隧道的最大容量
a1.channels.fileChannel.capacity = 10000
#-->事务容量的最大值设置
a1.channels.fileChannel.transactionCapacity = 200
```



kafka sink

```shell
#-->设置sources名称
a1.sources = sources1
#--> 设置channel名称
a1.channels = fileChannel
#--> 设置sink 名称
a1.sinks = sink1

# source 配置
a1.sources.sources1.type = TAILDIR
a1.sources.sources1.positionFile = /tmp/flume/taildir_position.json
a1.sources.sources1.filegroups = f1
a1.sources.sources1.filegroups.f1 = /u01/app/oracle/diag/rdbms/orcl/orcl/trace/alert_.*.log
a1.sources.sources1.batchSize = 100
a1.sources.sources1.backoffSleepIncrement = 1000
a1.sources.sources1.maxBackoffSleep = 5000
a1.sources.sources1.channels = fileChannel
a1.sources.sources1.headers.f1.headerKey = ora31
a1.sources.sources1.fileHeader = true


# sink1 配置
a1.sinks.sink1.channel = fileChannel
a1.sinks.sink1.type = org.apache.flume.sink.kafka.KafkaSink
a1.sinks.sink1.kafka.topic = mytopic
a1.sinks.sink1.kafka.bootstrap.servers = 10.117.130.146:9092
a1.sinks.sink1.kafka.flumeBatchSize = 100
a1.sinks.sink1.kafka.producer.acks = 1
a1.sinks.sink1.kafka.producer.linger.ms = 1
a1.sinks.sink1.kafka.producer.compression.type = snappy

# fileChannel 配置
a1.channels.fileChannel.type = file
#-->检测点文件所存储的目录
a1.channels.fileChannel.checkpointDir = /tmp/flume/checkpoint/oracle_alert/
#-->数据存储所在的目录设置
a1.channels.fileChannel.dataDirs = /tmp/flume/data/oracle_alert/
#-->隧道的最大容量
a1.channels.fileChannel.capacity = 10000
#-->事务容量的最大值设置
a1.channels.fileChannel.transactionCapacity = 200
```



kafka channel

```shell
#-->设置sources名称
a1.sources = sources1
#--> 设置channel名称
a1.channels = channel1
#--> 设置sink 名称
a1.sinks = sink1

# source 配置
a1.sources.sources1.type = TAILDIR
a1.sources.sources1.positionFile = /tmp/flume/taildir_position.json
a1.sources.sources1.filegroups = f1
a1.sources.sources1.filegroups.f1 = /u01/app/oracle/diag/rdbms/orcl/orcl/trace/alert_.*.log
a1.sources.sources1.batchSize = 100
a1.sources.sources1.backoffSleepIncrement = 1000
a1.sources.sources1.maxBackoffSleep = 5000
a1.sources.sources1.channels = channel1
a1.sources.sources1.headers.f1.headerKey = ora31
a1.sources.sources1.fileHeader = true


# sink1 配置
a1.sinks.sink1.channel = channel1
a1.sinks.sink1.type = org.apache.flume.sink.kafka.KafkaSink
a1.sinks.sink1.kafka.topic = mytopic
a1.sinks.sink1.kafka.bootstrap.servers = node02:9092
a1.sinks.sink1.kafka.flumeBatchSize = 100
a1.sinks.sink1.kafka.producer.acks = 1
a1.sinks.sink1.kafka.producer.linger.ms = 1
a1.sinks.sink1.kafka.producer.compression.type = snappy

# kafka channel 配置
a1.channels.channel1.type = org.apache.flume.channel.kafka.KafkaChannel
a1.channels.channel1.kafka.bootstrap.servers = node02:9092
a1.channels.channel1.kafka.topic = channel2
a1.channels.channel1.kafka.consumer.group.id = flume-consumer
```



拦截器

```shell
#-->设置sources名称
a1.sources = sources1
#--> 设置channel名称
a1.channels = channel1
#--> 设置sink 名称
a1.sinks = sink1

# source 配置
a1.sources.sources1.type = TAILDIR
a1.sources.sources1.positionFile = /tmp/flume/taildir_position.json
a1.sources.sources1.filegroups = f1
a1.sources.sources1.filegroups.f1 = /u01/app/oracle/diag/rdbms/orcl/orcl/trace/alert_.*.log
a1.sources.sources1.batchSize = 100
a1.sources.sources1.backoffSleepIncrement = 1000
a1.sources.sources1.maxBackoffSleep = 5000
a1.sources.sources1.channels = channel1
a1.sources.sources1.headers.f1.headerKey = ora31
a1.sources.sources1.fileHeader = true
# 设置拦截器
a1.sources.sources1.interceptors = i1
a1.sources.sources1.interceptors.i1.type = regex_filter
a1.sources.sources1.interceptors.i1.regex = ^ORA-.*
a1.sources.sources1.interceptors.i1.excludeEvents = false


# sink1 配置
a1.sinks.sink1.channel = channel1
a1.sinks.sink1.type = org.apache.flume.sink.kafka.KafkaSink
a1.sinks.sink1.kafka.topic = mytopic
a1.sinks.sink1.kafka.bootstrap.servers = node02:9092
a1.sinks.sink1.kafka.flumeBatchSize = 100
a1.sinks.sink1.kafka.producer.acks = 1
a1.sinks.sink1.kafka.producer.linger.ms = 1
a1.sinks.sink1.kafka.producer.compression.type = snappy

# kafka channel 配置
a1.channels.channel1.type = org.apache.flume.channel.kafka.KafkaChannel
a1.channels.channel1.kafka.bootstrap.servers = node02:9092
a1.channels.channel1.kafka.topic = channel2
a1.channels.channel1.kafka.consumer.group.id = flume-consumer

```



es

```shell
#-->设置sources名称
a1.sources = sources1
#--> 设置channel名称
a1.channels = channel1
#--> 设置sink 名称
a1.sinks = sink1

# source 配置
a1.sources.sources1.type = TAILDIR
a1.sources.sources1.positionFile = /tmp/flume/taildir_position.json
a1.sources.sources1.filegroups = f1
a1.sources.sources1.filegroups.f1 = /u01/app/oracle/diag/rdbms/orcl/orcl/trace/alert_.*.log
a1.sources.sources1.batchSize = 100
a1.sources.sources1.backoffSleepIncrement = 1000
a1.sources.sources1.maxBackoffSleep = 5000
a1.sources.sources1.channels = channel1
a1.sources.sources1.headers.f1.headerKey = ora31
a1.sources.sources1.fileHeader = true


# sink1 配置
a1.channels = channel1
a1.sinks = sink1
a1.sinks.sink1.type = elasticsearch
a1.sinks.sink1.hostNames = 10.117.130.176:9300 #这里不能按照文档写9200、9300
a1.sinks.sink1.indexName = foo_index
a1.sinks.sink1.indexType = bar_type
a1.sinks.sink1.clusterName = foobar_cluster
a1.sinks.sink1.batchSize = 500
a1.sinks.sink1.ttl = 5d
a1.sinks.sink1.serializer = org.apache.flume.sink.elasticsearch.ElasticSearchDynamicSerializer
a1.sinks.sink1.channel = channel1
	
# kafka channel 配置
a1.channels.channel1.type = org.apache.flume.channel.kafka.KafkaChannel
a1.channels.channel1.kafka.bootstrap.servers = node02:9092
a1.channels.channel1.kafka.topic = channel2
a1.channels.channel1.kafka.consumer.group.id = flume-consumer
```



oom

调整flume-ng 启动文件里面有关于内存的参数