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



内部有三个组件：

1. Source：采集源，用于跟数据源对接，以获取数据
2. Sink：下沉地，采集数据的传送目的，用于往下一级agent传递数据或者往最终存储系统传递数据
3. Channel：angent内部的数据传输通道，用于从source将数据传递到sink



```bash
# 基本配置
example.conf

# 从日志获取
exec-memory-logger.conf

# 数据传递，a机器发送到b机器接收，然后输出到logger
exec-memory-avro.conf
avro-memory-logger.conf
```



启动方法

```bash
$ bin/flume-ng agent --conf $FLUME_HOME/conf \
--conf-file $FLUME_HOME/conf/example.conf \
--name a1 \
-Dflume.root.logger=INFO,console
```


