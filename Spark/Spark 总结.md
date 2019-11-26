##  Spark 总结

[TOC]

### 算子的选择

#### map vs mapPartitions

1. `Return a new RDD by applying a function to all elements of this RDD`作用在每一个元素上面

2. `Return a new RDD by applying a function to each partition of this RDD`作用在每一个partition上面

3. 例子把数据写到MySQL中使用，如果使用map会导致开启过多的链接，使用partition会已分区为单位

   ```scala
     def myMap(rdd: RDD[String]): Unit = {
       rdd.map(x => {
         val connection = MySqlPool.getJdbcConn()
         println(connection + "~~~~~")
   
         // TODO: 保存数据导db
   
         MySqlPool.releaseConn(connection)
       }).foreach(println)
     }
   
   
     def myMapPartition(rdd: RDD[String]): Unit = {
       rdd.mapPartitions(partition=>{
         val connection = MySqlPool.getJdbcConn()
         println(connection + "~~~~~")
   
         // TODO: 保存数据导db
   
         MySqlPool.releaseConn(connection)
         partition
       }).foreach(println)
     }
   ```



#### foreach vs foreachPartition

1. 和上面例子原理一样， foreach是算子操作。 

   例子

   ```scala
     def myForeach(rdd: RDD[String]): Unit = {
       rdd.foreach(x => {
         val connection = MySqlPool.getJdbcConn()
         println(connection + "~~~~~")
   
         // TODO: 保存数据导db
   
         MySqlPool.releaseConn(connection)
       })
     }
   
   
     def myForeachPartition(rdd: RDD[String]): Unit = {
       rdd.foreachPartition(partition => {
         val connection = MySqlPool.getJdbcConn()
         println(connection + "~~~~~")
   
         // TODO: 保存数据导db
   
         MySqlPool.releaseConn(connection)
         partition
       })
     }
   ```

#### reduceByKey vs groupByKey

![](https://raw.githubusercontent.com/jacksu/utils4s/master/spark-knowledge/images/reduceByKey.png)

![](https://raw.githubusercontent.com/jacksu/utils4s/master/spark-knowledge/images/groupByKey.png)

1. reduceByKey会在本地进行预聚合，然后将集合的数据进行shuffle操作。

2. groupByKey如图所示， 没有进行预聚合直接shuffle。

   示例

```scala
    // reduceByKey
    sc.textFile("file:////root/wangxi/test_tb1.txt")
      .flatMap(_.split("\t"))
      .map((_, 1))
      .reduceByKey(_ + _)
      .collect()

    // groupByKey
    sc.textFile("file:////root/wangxi/test_tb1.txt")
      .flatMap(_.split("\t"))
      .map((_, 1))
      .groupByKey()
      .map(x => (x._1, x._2.sum))
      .collect()
```



#### collect

1. ```scala
   /**
    * Return an array that contains all of the elements in this RDD.
    *
    * @note This method should only be used if the resulting array is expected to be small, as
    * all the data is loaded into the driver's memory.
    */
   ```

1. 执行结果全部放到一个数组里，内存不足会造成OOM。
2. 可以使用take(N)查看数据，或者save到hdfs里在进行查看。



#### coalesce vs repartition

1. coalesce减少partition不会进行shuffle。
2. repartition增大、减少partition都会进行shuffle。

```scala
  def coalesce(numPartitions: Int, shuffle: Boolean = false,
               partitionCoalescer: Option[PartitionCoalescer] = Option.empty)
              (implicit ord: Ordering[T] = null)
      : RDD[T] = withScope {
    require(numPartitions > 0, s"Number of partitions ($numPartitions) must be positive.")
    if (shuffle) {
      /** Distributes elements evenly across output partitions, starting from a random partition. */
      val distributePartition = (index: Int, items: Iterator[T]) => {
        var position = new Random(hashing.byteswap32(index)).nextInt(numPartitions)
        items.map { t =>
          // Note that the hash code of the key will just be the key itself. The HashPartitioner
          // will mod it with the number of total partitions.
          position = position + 1
          (position, t)
        }
      } : Iterator[(Int, T)]


def repartition(numPartitions: Int)(implicit ord: Ordering[T] = null): RDD[T] = withScope {
    coalesce(numPartitions, shuffle = true)
  }
```



### 序列化的选择

#### Serialization Java & Kryo

| ID   | RDD Name              | Storage Level                     | Cached Partitions | Fraction Cached | Size in Memory |                     |
| :--- | :-------------------- | :-------------------------------- | :---------------- | :-------------- | :------------- | ------------------- |
| 0    | ParallelCollectionRDD | Memory Serialized 1x Replicated   | 2                 | 100%            | 25.3 MB        | Java  serialization |
| 1    | ParallelCollectionRDD | Memory Deserialized 1x Replicated | 2                 | 100%            | 34.3 MB        | Java memory_only    |
| 2    | ParallelCollectionRDD | Memory Serialized 1x Replicated   | 2                 | 100%            | 68.1 MB        | Kryo(未注册)        |
| 3    | ParallelCollectionRDD | Memory Serialized 1x Replicated   | 2                 | 100%            | 26.1 MB        | Kryo(注册)          |

1. Kryo在未使用注册的情况下，内存占用更多。 需要特别注意。



### sink to MySQL

#### 错误的使用示例

```SCALA
dstream.foreachRDD { rdd =>
  val connection = createNewConnection()  // executed at the driver
  rdd.foreach { record =>
    connection.send(record) // executed at the worker
  }
}
```

#### 正确的使用示例

```scala
// TODO: 需要从socket接收的数据进行处理之后写入MySQL
    val result = lines.flatMap(_.split(",")).map((_, 1)).reduceByKey(_ + _)

    result.foreachRDD(rdd => {
      rdd.foreachPartition(partitionOfRecords => {
        val connection = MySqlPool.getJdbcConn()
        partitionOfRecords.foreach(record => {
          val sql = s"insert into wc(word,count) values('${record._1}', ${record._2})"
          connection.createStatement().execute(sql)
        })
        MySqlPool.releaseConn(connection)
      })
    })
```



### Spark常见内容

#### Spark on YARN两种方式的区别已经工作流程

![](https://hadoop.apache.org/docs/r2.7.7/hadoop-yarn/hadoop-yarn-site/yarn_architecture.gif)

**Yarn-cluster模式下作业执行流程：**

1. 客户端生成作业信息提交给ResourceManager(RM)
2. RM在某一个NodeManager(由Yarn决定)启动container并将Application Master(AM)分配给该NodeManager(NM)
3. NM接收到RM的分配，启动Application Master并初始化作业，此时这个NM就称为Driver
4. Application向RM申请资源，分配资源同时通知其他NodeManager启动相应的Executor

**Yarn-client模式下作业执行流程：**

1. 客户端生成作业信息提交给ResourceManager(RM)
2. RM在本地NodeManager启动container并将Application Master(AM)分配给该NodeManager(NM)
3. NM接收到RM的分配，启动Application Master并初始化作业，此时这个NM就称为Driver
4. Application向RM申请资源，分配资源同时通知其他NodeManager启动相应的Executor
5. Executor向本地启动的Application Master注册汇报并完成相应的任务



#### Spark内存管理

![](http://ww1.sinaimg.cn/large/6e46250bly1g1n6kgyy12j20lc0mgdgp.jpg)

Spark 2.1.0 新型 JVM Heap 分成三个部份：Reserved Memory、User Memory 和 Spark Memory。

- **Reserved Memory**：默认都是300MB，这个数字一般都是固定不变的，在系统运行的时候 Java Heap 的大小至少为 **Heap Reserved Memory x 1.5**. e.g. 300MB x 1.5 = 450MB 的 JVM配置。一般本地开发例如说在 Windows 系统上，建义系统至少 2G 的大小。
- **User Memory**：写 Spark 程序中产生的临时数据或者是自己维护的一些数据结构也需要给予它一部份的存储空间，你可以这么认为，这是程序运行时用户可以主导的空间，叫用户操作空间。它占用的空间是 **(Java Heap - Reserved Memory) x 25%** (默认是25％，可以有参数供调优)，这样设计可以让用户操作时所需要的空间与系统框架运行时所需要的空间分离开。假设 Executor 有 4G 的大小，那么在默认情况下 User Memory 大小是：(4G - 300MB) x 25% = 949MB，也就是说一个 Stage 内部展开后 Task 的算子在运行时最大的大小不能够超过 949MB。例如工程师使用 mapPartition 等，一个 Task 内部所有算子使用的数据空间的大小如果大于 949MB 的话，那么就会出现 OOM。思考题：有 100个 Executors 每个 4G 大小，现在要处理 100G 的数据，假设这 100G 分配给 100个 Executors，每个 Executor 分配 1G 的数据，这 1G 的数据远远少于 4G Executor 内存的大小，为什么还会出现 OOM 的情况呢？那是因为在你的代码中(e.g.你写的应用程序算子）超过用户空间的限制 (e.g. 949MB)，而不是 RDD 本身的数据超过了限制。
- **Spark Memeory**：系统框架运行时需要使用的空间，这是从两部份构成的，分别是 Storage Memeory 和 Execution Memory。现在 Storage 和 Execution (Shuffle) 采用了 Unified 的方式共同使用了 (Heap Size - 300MB) x 75%，默认情况下 Storage 和 Execution 各占该空间的 50%。你可以从图中可以看出，Storgae 和 Execution 的存储空间可以往上和往下移动。
  定义：所谓 Unified 的意思是 Storgae 和 Execution 在适当时候可以借用彼此的 Memory，需要注意的是，当 Execution 空间不足而且 Storage 空间也不足的情况下，Storage 空间如果曾经使用了超过 Unified 默认的 50% 空间的话则超过部份会被强制 drop 掉一部份数据来解决 Execution 空间不足的问题 (注意：drop 后数据会不会丢失主要是看你在程序设置的 storage_level 来决定你是 Drop 到那里，可能 Drop 到磁盘上)，这是因为执行(Execution) 比缓存 (Storage) 是更重要的事情。

| Property Name                  | Default | Meaning                                                      |
| :----------------------------- | :------ | :----------------------------------------------------------- |
| `spark.memory.fraction`        | 0.6     | Fraction of (heap space - 300MB) used for execution and storage. The lower this is, the more frequently spills and cached data eviction occur. The purpose of this config is to set aside memory for internal metadata, user data structures, and imprecise size estimation in the case of sparse, unusually large records. Leaving this at the default value is recommended. For more detail, including important information about correctly tuning JVM garbage collection when increasing this value, see [this description](http://spark.apache.org/docs/latest/tuning.html#memory-management-overview). |
| `spark.memory.storageFraction` | 0.5     | Amount of storage memory immune to eviction, expressed as a fraction of the size of the region set aside by `spark.memory.fraction`. The higher this is, the less working memory may be available to execution and tasks may spill to disk more often. Leaving this at the default value is recommended. For more detail, see [this description](http://spark.apache.org/docs/latest/tuning.html#memory-management-overview). |



#### Spark作业资源的设置情况 executor、memory、core、drive、task、partition

- Spark 在分配内存时，会在用户设定的内存值上溢出 375M 或 7%（取大值）。

- Yarn 分配 container 内存时，遵循向上取整的原则，这里也就是需要满足 1G 的整数倍。

- | 节点   | 资源类型 | 资源量（结果使用上面的例子计算得到） |
  | :----- | :------- | :----------------------------------- |
  | master | core     | 1                                    |
  |        | mem      | driver-memroy = 4G                   |
  | worker | core     | num-executors * executor-cores = 4   |
  |        | mem      | num-executors * executor-memory = 4G |

- 指定并行的task数量`spark.default.parallelism`设置该参数为num-executors * executor-cores的2~3倍较为合适

- 指定partition数量`spark.sql.shuffle.partitions`

```shell
spark-submit \
--class com.vincent.interview.SparkHiveExample2 \
--master yarn \
--deploy-mode cluster \
--driver-memory 1g \
--executor-cores 2 \
--executor-memory 2g \
--num-executors 4 \
--conf spark.default.parallelism=1000 \ #设置task
--conf spark.executor.memoryOverhead=2048 \ #设置堆外内存Reserved Memory
--conf spark.network.timeout=300 \ #设置超时时间
/root/wangxi/lib/sparktest-1.0-SNAPSHOT.jar
```



#### Shuffle的机制 shuffle、依赖

- Spark 0.8及以前 Hash Based Shuffle
- Spark 0.8.1 为Hash Based Shuffle引入File Consolidation机制
- Spark 0.9 引入ExternalAppendOnlyMap
- Spark 1.1 引入Sort Based Shuffle，但默认仍为Hash Based Shuffle
- Spark 1.2 默认的Shuffle方式改为Sort Based Shuffle
- Spark 1.4 引入Tungsten-Sort Based Shuffle
- Spark 1.6 Tungsten-sort并入Sort Based Shuffle
- Spark 2.0 Hash Based Shuffle退出历史舞台

总结一下， 就是最开始的时候使用的是 Hash Based Shuffle， 这时候每一个Mapper会根据Reducer的数量创建出相应的bucket，bucket的数量是M *R ，*其中M是Map的个数，R是Reduce的个数。这样会产生大量的小文件，对文件系统压力很大，而且也不利于IO吞吐量。后面忍不了了就做了优化，把在同一core上运行的多个Mapper 输出的合并到同一个文件，这样文件数目就变成了 cores R 个了



**Hash Based Shuffle**

3个 map task， 3个 reducer， 会产生 9个小文件，

![](http://ww1.sinaimg.cn/large/6e46250bly1g1o26vo72lj20hs06vdi8.jpg)



开启Consolidation改造之后

4个map task， 4个reducer， 如果不使用 Consolidation机制， 会产生 16个小文件。

但是但是现在这 4个 map task 分两批运行在 2个core上， 这样只会产生 8个小文件

![](http://ww1.sinaimg.cn/large/6e46250bly1g1o27y5xmzj20hs07j0x8.jpg)



**Sort Based Shuffle**

BypassMergeSortShuffleWriter

BypassMergeSortShuffleWriter和Hash Shuffle中的HashShuffleWriter实现基本一致， 唯一的区别在于，map端的多个输出文件会被汇总为一个文件。 所有分区的数据会合并为同一个文件，会生成一个索引文件，是为了索引到每个分区的起始地址，可以随机 access 某个partition的所有数据。

![](http://ww1.sinaimg.cn/large/6e46250bly1g1o32wl4inj20hs0brada.jpg)

#### 数据倾斜

1. 可以尝试增加task
2. 查看map源数据是否为可分割的。如lzo.index
3.  使用Broadcast join 
   1. `SELECT /*+ BROADCAST(r) */ * FROM records r JOIN src s ON r.key = s.key`
   2. `SET spark.sql.autoBroadcastJoinThreshold=104857600;`
4. 为skew的key增加随机前/后缀
5. Adaptive Execution

```shell
set spark.sql.adaptive.enabled=true;

--开启 Adaptive Execution 的动态调整 Join 功能
set spark.sql.adaptive.join.enabled=true;

--设置为 true 即可自动处理 Join 时数据倾斜
set spark.sql.adaptive.skewedJoin.enabled=true;

--控制处理一个倾斜 Partition 的 Task 个数上限，默认值为 5
set spark.sql.adaptive.skewedPartitionMaxSplits=30; 
```

6. 使用rbo

```
set spark.sql.cbo=true;
set spark.sql.statistics.histogram.enabled=true;

ANALYZE TABLE table_name COMPUTE STATISTICS

ANALYZE TABLE table_name COMPUTE STATISTICS FOR COLUMNS column-name1, column-name2, ….
```



#### Catalyst的流程

![](http://ww1.sinaimg.cn/large/6e46250bly1g1pgldjwy0j215d0j3tbu.jpg)

从上图可见，无论是直接使用 SQL 语句还是使用 DataFrame，都会经过如下步骤转换成 DAG 对 RDD 的操作

- Parser 解析 SQL，生成 Unresolved Logical Plan
- 由 Analyzer 结合 Catalog 信息生成 Resolved Logical Plan
- Optimizer根据预先定义好的规则对 Resolved Logical Plan 进行优化并生成 Optimized Logical Plan
- Query Planner 将 Optimized Logical Plan 转换成多个 Physical Plan
- CBO 根据 Cost Model 算出每个 Physical Plan 的代价并选取代价最小的 Physical Plan 作为最终的 Physical Plan
- Spark 以 DAG 的方法执行上述 Physical Plan
- 在执行 DAG 的过程中，Adaptive Execution 根据运行时信息动态调整执行计划从而提高执行效率



#### Spark和MapReduce相比都有哪些优势？

传统的MapReduce虽然具有自动容错、平衡负载和可拓展性的优点，但是其最大缺点是采用非循环式的数据流模型（由于每一次MapReduce的输入/输出数据，都需要读取/写入磁盘当中，如果涉及到多个作业流程，就意味着多次读取和写入HDFS），使得在迭代计算式要进行大量的磁盘IO操作。

RDD抽象出一个被分区、不可变、且能并行操作的数据集；从HDFS读取的需要计算的数据，在经过处理后的中间结果会作为RDD单元缓存到内存当中，并可以作为下一次计算的输入信息。最终Spark只需要读取和写入一次HDFS，这样就避免了Hadoop MapReduce的大IO操作。



#### DataFrame/Dataset/RDD

DataFrame 分布式的数据集

A Dataset is a distributed collection of data 

以列（列名、列的类型、列值）的形式构成的分布式数据集，按照列赋予不同的名称

A DataFrame is a Dataset organized into named columns. 

Spark提供的主要抽象是*弹性分布式数据集*（RDD），它是跨群集节点分区的元素的集合，可以并行操作







1. Spark作业执行流程：count 后续做了什么。 

2. Spark隐式转换的作用

3. Spark的规模

4. Spark OOM如何解决

5. ThriftServer如何实现HA

6. Kafka整合Spark offset管理

7. jvm内存重用

   ```
   mapreduce.job.jvm.numtasks
   ```

   

hadoop dfsadmin -safemode leave 



```
spark-sql --master yarn --executor-cores 6 --executor-memory 6g --num-executors 4 --driver-class-path /usr/local/hadoop/share/hadoop/common/hadoop-lzo-0.4.21-SNAPSHOT.jar

```

