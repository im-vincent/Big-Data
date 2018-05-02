## 重要概念

1. **Transformation** 不实际运算、记录元数据
2. **Action** 遇到action真正运算
3. **job** 每个action对应一个job
4. **stage** 是否shuffle作为界定依据

groupByKey

![](http://ww1.sinaimg.cn/large/6e46250bly1fknb9mdoedj20yt0ekq5h.jpg)

reduceByKey

![](http://ww1.sinaimg.cn/large/6e46250bly1fknc0reb26j20yx0eowhg.jpg)



没有开启consolidation

![](http://ww1.sinaimg.cn/large/6e46250bly1fkncb4l712j20z30ermzk.jpg)



开启consolidateion 

```
conf.set("spark.shuffle.consolidateFiles", "true");
```

![](http://ww1.sinaimg.cn/large/6e46250bly1fknco4e081j20z50eo76b.jpg)



spark.reducer.maxMbInFlight





使用spark2-submit运行java

```shell
spark2-submit --class org.apache.spark.SparkSQLOnHive hadoopsparkjava.jar --master yarn --deploy-mode cluster
```



提交Spark App到环境中运行

```bash
spark2-submit \
  --name SQLContextApp \
  --class com.imooc.spark.SQLContextApp \
  --master local[2] \
  --jars /usr/share/java/mysql-connector-java.jar \
  /root/lib/sql-1.0.jar \
  /root/resource/people.json
```

```
spark2-submit --name Record  \
--class cn.com.cis.Record   \
--master yarn \
--num-executors 8 \
--executor-memory 1G \
--conf spark.shuffle.consolidateFiles=true \
--conf spark.default.parallelism=100 \
--conf spark.storage.memoryFraction=0.5 \
--conf spark.shuffle.memoryFraction=0.3 \
/root/wangxi/spark-1.0.jar


```



控制台使用spark sql

```sql
scala> spark.sql("show tables").show
+--------+---------------+-----------+
|database|      tableName|isTemporary|
+--------+---------------+-----------+
| default|      accesslog|      false|
| default|    tb_emp_info|      false|
| default|          trlog|      false|
| default|   user_address|      false|
| default|user_basic_info|      false|
| default|      user_info|      false|
| default|            web|      false|
+--------+---------------+-----------+
```



查看系统函数

```sql
scala> spark.sql("show functions").show(10);
+--------+
|function|
+--------+
|       !|
|       %|
|       &|
|       *|
|       +|
|       -|
|       /|
|       <|
|      <=|
|     <=>|
+--------+
only showing top 10 rows
```



## 宽依赖与窄依赖

- 窄依赖是指父RDD的每个分区只被子RDD的一个分区所使用，**子RDD分区通常对应常数个父RDD分区**(O(1)，与数据规模无关)
- 相应的，宽依赖是指父RDD的每个分区都可能被多个子RDD分区所使用，**子RDD分区通常对应所有的父RDD分区**(O(n)，与数据规模有关)

宽依赖和窄依赖如下图所示：

![宽依赖和窄依赖示例](https://img-blog.csdn.net/20160913233559680)

相比于宽依赖，窄依赖对优化很有利 ，主要基于以下两点：

1. 宽依赖往往对应着shuffle操作，需要在运行过程中将同一个父RDD的分区传入到不同的子RDD分区中，中间可能涉及多个节点之间的数据传输；而窄依赖的每个父RDD的分区只会传入到一个子RDD分区中，通常可以在一个节点内完成转换。

2. 当RDD分区丢失时（某个节点故障），spark会对数据进行重算。

   1. 对于窄依赖，由于父RDD的一个分区只对应一个子RDD分区，这样只需要重算和子RDD分区对应的父RDD分区即可，所以这个重算对数据的利用率是100%的；

   2. 对于宽依赖，重算的父RDD分区对应多个子RDD分区，这样实际上父RDD 中只有一部分的数据是被用于恢复这个丢失的子RDD分区的，另一部分对应子RDD的其它未丢失分区，这就造成了多余的计算；更一般的，宽依赖中子RDD分区通常来自多个父RDD分区，极端情况下，所有的父RDD分区都要进行重新计算。

   3. 如下图所示，b1分区丢失，则需要重新计算a1,a2和a3，这就产生了冗余计算(a1,a2,a3中对应b2的数据)。

      ![宽依赖](https://img-blog.csdn.net/20160913234059245)

以下是文章 [RDD：基于内存的集群计算容错抽象](http://shiyanjun.cn/archives/744.html) 中对宽依赖和窄依赖的对比。

> 区分这两种依赖很有用。首先，窄依赖允许在一个集群节点上以流水线的方式（pipeline）计算所有父分区。例如，逐个元素地执行map、然后filter操作；而宽依赖则需要首先计算好所有父分区数据，然后在节点之间进行Shuffle，这与MapReduce类似。第二，窄依赖能够更有效地进行失效节点的恢复，即只需重新计算丢失RDD分区的父分区，而且不同节点之间可以并行计算；而对于一个宽依赖关系的Lineage图，单个节点失效可能导致这个RDD的所有祖先丢失部分分区，因而需要整体重新计算。

【误解】之前一直理解错了，以为窄依赖中每个子RDD可能对应多个父RDD,当子RDD丢失时会导致多个父RDD进行重新计算，所以窄依赖不如宽依赖有优势。**而实际上应该深入到分区级别去看待这个问题，而且重算的效用也不在于算的多少，而在于有多少是冗余的计算。窄依赖中需要重算的都是必须的，所以重算不冗余。**

窄依赖的函数有：map, filter, union, join(父RDD是hash-partitioned ), mapPartitions, mapValues 
宽依赖的函数有：groupByKey, join(父RDD不是hash-partitioned ), partitionBy





## Yarn-cluster VS Yarn-client

当在Spark On Yarn模式下，每个Spark Executor作为一个Yarn container在运行，同时支持多个任务在同一个container中运行，极大地节省了任务的启动时间

 

**Appliaction Master**

为了更好的理解这两种模式的区别先了解下Yarn的Application Master概念，在Yarn中，每个application都有一个Application Master进程，它是Appliaction启动的第一个容器，它负责从ResourceManager中申请资源，分配资源，同时通知NodeManager来为Application启动container，Application Master避免了需要一个活动的client来维持，启动Applicatin的client可以随时退出，而由Yarn管理的进程继续在集群中运行

------

**Yarn-cluster**

在Yarn-cluster模式下，driver运行在Appliaction Master上，Appliaction Master进程同时负责驱动Application和从Yarn中申请资源，该进程运行在Yarn container内，所以启动Application Master的client可以立即关闭而不必持续到Application的生命周期，下图是yarn-cluster模式

![img](https://images2015.cnblogs.com/blog/776259/201609/776259-20160909165742332-1159110454.png)

**Yarn-cluster模式下作业执行流程：**

1. 客户端生成作业信息提交给ResourceManager(RM)
2. RM在某一个NodeManager(由Yarn决定)启动container并将Application Master(AM)分配给该NodeManager(NM)
3. NM接收到RM的分配，启动Application Master并初始化作业，此时这个NM就称为Driver
4. Application向RM申请资源，分配资源同时通知其他NodeManager启动相应的Executor

**Yarn-client**

在Yarn-client中，Application Master仅仅从Yarn中申请资源给Executor，之后client会跟container通信进行作业的调度，下图是Yarn-client模式

![img](https://images2015.cnblogs.com/blog/776259/201609/776259-20160909165804926-17816733.png)

**Yarn-client模式下作业执行流程：**

1. 客户端生成作业信息提交给ResourceManager(RM)
2. RM在本地NodeManager启动container并将Application Master(AM)分配给该NodeManager(NM)
3. NM接收到RM的分配，启动Application Master并初始化作业，此时这个NM就称为Driver
4. Application向RM申请资源，分配资源同时通知其他NodeManager启动相应的Executor
5. Executor向本地启动的Application Master注册汇报并完成相应的任务

 ![img](file:///D:/%E4%B8%BA%E7%9F%A5%E7%AC%94%E8%AE%B0%E6%95%B0%E6%8D%AE/temp/87e4b7ac-f143-47b6-9b6e-20b5ea662ce9/128/index_files/de48a141-d70f-4b25-b1ea-08f2a17a4ea6.png)![img](https://images2015.cnblogs.com/blog/776259/201609/776259-20160909165822723-1513641104.png)



## Spark作业基本运行原理

![](https://img-blog.csdn.net/20160515225532299)

### 1.num-executors

- 参数说明：该参数用于设置Spark作业总共要用多少个Executor进程来执行。Driver在向YARN集群管理器申请资源时，YARN集群管理器会尽可能按照你的设置来在集群的各个工作节点上，启动相应数量的Executor进程。这个参数非常之重要，如果不设置的话，默认只会给你启动少量的Executor进程，此时你的Spark作业的运行速度是非常慢的。
- 参数调优建议：每个Spark作业的运行一般设置50~100个左右的Executor进程比较合适，设置太少或太多的Executor进程都不好。设置的太少，无法充分利用集群资源；设置的太多的话，大部分队列可能无法给予充分的资源。

### 2.executor-memory

- 参数说明：该参数用于设置每个Executor进程的内存。Executor内存的大小，很多时候直接决定了Spark作业的性能，而且跟常见的JVM OOM异常，也有直接的关联。
- 参数调优建议：每个Executor进程的内存设置4G~8G较为合适。但是这只是一个参考值，具体的设置还是得根据不同部门的资源队列来定。可以看看自己团队的资源队列的最大内存限制是多少，num-executors乘以executor-memory，是不能超过队列的最大内存量的。此外，如果你是跟团队里其他人共享这个资源队列，那么申请的内存量最好不要超过资源队列最大总内存的1/3~1/2，避免你自己的Spark作业占用了队列所有的资源，导致别的同学的作业无法运行。

### 3.executor-cores

- 参数说明：该参数用于设置每个Executor进程的CPU core数量。这个参数决定了每个Executor进程并行执行task线程的能力。因为每个CPU core同一时间只能执行一个task线程，因此每个Executor进程的CPU core数量越多，越能够快速地执行完分配给自己的所有task线程。
- 参数调优建议：Executor的CPU core数量设置为2~4个较为合适。同样得根据不同部门的资源队列来定，可以看看自己的资源队列的最大CPU core限制是多少，再依据设置的Executor数量，来决定每个Executor进程可以分配到几个CPU core。同样建议，如果是跟他人共享这个队列，那么num-executors * executor-cores不要超过队列总CPU core的1/3~1/2左右比较合适，也是避免影响其他同学的作业运行。

### 4.driver-memory

- 参数说明：该参数用于设置Driver进程的内存。
- 参数调优建议：Driver的内存通常来说不设置，或者设置1G左右应该就够了。唯一需要注意的一点是，如果需要使用collect算子将RDD的数据全部拉取到Driver上进行处理，那么必须确保Driver的内存足够大，否则会出现OOM内存溢出的问题。

### 5.spark.default.parallelism

- 参数说明：该参数用于设置每个stage的默认task数量。这个参数极为重要，如果不设置可能会直接影响你的Spark作业性能。
- 参数调优建议：Spark作业的默认task数量为500~1000个较为合适。很多同学常犯的一个错误就是不去设置这个参数，那么此时就会导致Spark自己根据底层HDFS的block数量来设置task的数量，默认是一个HDFS block对应一个task。通常来说，Spark默认设置的数量是偏少的（比如就几十个task），如果task数量偏少的话，就会导致你前面设置好的Executor的参数都前功尽弃。试想一下，无论你的Executor进程有多少个，内存和CPU有多大，但是task只有1个或者10个，那么90%的Executor进程可能根本就没有task执行，也就是白白浪费了资源！因此Spark官网建议的设置原则是，设置该参数为num-executors * executor-cores的2~3倍较为合适，比如Executor的总CPU core数量为300个，那么设置1000个task是可以的，此时可以充分地利用Spark集群的资源。

### 6.spark.storage.memoryFraction

- 参数说明：该参数用于设置RDD持久化数据在Executor内存中能占的比例，默认是0.6。也就是说，默认Executor 60%的内存，可以用来保存持久化的RDD数据。根据你选择的不同的持久化策略，如果内存不够时，可能数据就不会持久化，或者数据会写入磁盘。
- 参数调优建议：如果Spark作业中，有较多的RDD持久化操作，该参数的值可以适当提高一些，保证持久化的数据能够容纳在内存中。避免内存不够缓存所有的数据，导致数据只能写入磁盘中，降低了性能。但是如果Spark作业中的shuffle类操作比较多，而持久化操作比较少，那么这个参数的值适当降低一些比较合适。此外，如果发现作业由于频繁的gc导致运行缓慢（通过spark web ui可以观察到作业的gc耗时），意味着task执行用户代码的内存不够用，那么同样建议调低这个参数的值。

### 7.spark.shuffle.memoryFraction

- 参数说明：该参数用于设置shuffle过程中一个task拉取到上个stage的task的输出后，进行聚合操作时能够使用的Executor内存的比例，默认是0.2。也就是说，Executor默认只有20%的内存用来进行该操作。shuffle操作在进行聚合时，如果发现使用的内存超出了这个20%的限制，那么多余的数据就会溢写到磁盘文件中去，此时就会极大地降低性能。
- 参数调优建议：如果Spark作业中的RDD持久化操作较少，shuffle操作较多时，建议降低持久化操作的内存占比，提高shuffle操作的内存占比比例，避免shuffle过程中数据过多时内存不够用，必须溢写到磁盘上，降低了性能。此外，如果发现作业由于频繁的gc导致运行缓慢，意味着task执行用户代码的内存不够用，那么同样建议调低这个参数的值。

### 8.资源参数参考示例

```shell
spark2-submit --name Record \
--class cn.com.cis.Record \
--master yarn \
--deploy-mode cluster \
--num-executors 3 \
--executor-cores 4 \
--executor-memory 1200m \
--conf spark.shuffle.consolidateFiles=true \
--conf spark.default.parallelism=1000 \
--conf spark.storage.memoryFraction=0.5 \
--conf spark.shuffle.memoryFraction=0.3 \
/root/wangxi/spark-1.0.jar
```



## Window Operations

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