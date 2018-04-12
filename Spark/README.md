#### Window Operations

![](https://spark.apache.org/docs/latest/img/streaming-dstream-window.png)

| 名字             | 解释                             |
| ---------------- | -------------------------------- |
| window           | 定时的进行一个时间段内的数据处理 |
| window  length   | 窗口的长度                       |
| sliding interval | 窗口的间隔                       |

这2个参数和我们的batch size有倍数关系

每隔多久计算某个范围内的数据  例：每隔10秒计算前10秒的wc

案例代码 TransformApp.scala



## Flume-style Push-based Approach

Due to the push model, the streaming application needs to be up, with the receiver scheduled and listening on the chosen port, for Flume to be able push data.

flume推送数据到spark，要先启动spark

`flume_push_streaming.conf`

```
 groupId = org.apache.spark
 artifactId = spark-streaming-flume_2.11
 version = 2.3.0
```

案例代码 FlumePushWordCount



## Pull-based Approach using a Custom Sink

#### Configuring Flume

Configuring Flume on the chosen machine requires the following two steps.

1. **Sink JARs**: Add the following JARs to Flume’s classpath (see [Flume’s documentation](https://flume.apache.org/documentation.html) to see how) in the machine designated to run the custom sink .

   (i) *Custom sink JAR*: Download the JAR corresponding to the following artifact (or [direct link](http://search.maven.org/remotecontent?filepath=org/apache/spark/spark-streaming-flume-sink_2.11/2.3.0/spark-streaming-flume-sink_2.11-2.3.0.jar)).

   ```
    groupId = org.apache.spark
    artifactId = spark-streaming-flume-sink_2.11
    version = 2.3.0

   ```

   (ii) *Scala library JAR*: Download the Scala library JAR for Scala 2.11.8. It can be found with the following artifact detail (or, [direct link](http://search.maven.org/remotecontent?filepath=org/scala-lang/scala-library/2.11.8/scala-library-2.11.8.jar)).

   ```
    groupId = org.scala-lang
    artifactId = scala-library
    version = 2.11.8

   ```

   (iii) *Commons Lang 3 JAR*: Download the Commons Lang 3 JAR. It can be found with the following artifact detail (or, [direct link](http://search.maven.org/remotecontent?filepath=org/apache/commons/commons-lang3/3.5/commons-lang3-3.5.jar)).

   ```
    groupId = org.apache.commons
    artifactId = commons-lang3
    version = 3.5

   ```

2. **Configuration file**: On that machine, configure Flume agent to send data to an Avro sink by having the following in the configuration file.

   `flume_pull_streaming.conf`

   案例代码 FlumePullWordCount