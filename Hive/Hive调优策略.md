### Hive调优策略

#### 架构层面

1. 分表

2. 分区表 partition

   ```
   单级分区/多集分区
   
   创建一个分区表，分区的单位时dt和国家名
   hive> create table logs(ts bigint,line string)
       > partitioned by (dt String,country string);
       
   hive> load data local inpath '/root/hive/partitions/file1' into table logs
       > partition (dt='2001-01-01',country='GB');
       
   查看分区结构
   hive> show partitions logs;
   OK
   dt=2001-01-01/country=GB
   dt=2001-01-01/country=US
   dt=2001-01-02/country=GB
   dt=2001-01-02/country=US
   
   
   静态分区/动态分区
   	关系型数据库（如Oracle）中，对分区表Insert数据时候，数据库自动会根据分区字段的值，将数据插入到相应的分区中，Hive中也提供了类似的机制，即动态分区(Dynamic Partition)，只不过，使用Hive的动态分区，需要进行相应的配置。
   ```

3. 充分利用中间结果集合

4. 压缩

![](http://ww1.sinaimg.cn/large/6e46250bly1g1bl33nxr7j20rh0ffgrx.jpg)

```
压缩： 使用压缩算法来“减少”数据的过程。 
解压缩： 将压缩后的数据转换成原始数据的过程

需要考虑的点：
1. 压缩比
2. 压缩、解压缩的速度
3. 压缩文件是否可切割

lzo压缩具有可分割、快速压缩等特点。 一般hadoop采用此种压缩算法。
```



#### 语法层面

1. 排序： order by/sort by/distribute by/cluster by

   ```
   order by 能保证全局排序，严格模式需要指定limit
   sort by 在一个reduce里保证排序
   distribute by 按照一个列名进行分发到不同hdfs文件中。
   cluster by = distribute by + sort by
   ```

2. 控制输出（reduce/partition/task）的数量

   ```
   reduce的数量代表数据落地文件的数量。 hive里是reduce、spark里是partition
   set mapred.reduce.tasks
   ```

3. join：普通join/mapjoin

   ```
   set hive.auto.convert.join;
   map join执行步骤：
   本地工作：在本地机器读入数据，在内存中建立hash表，然后写入本地磁盘并上传至hdfs。最后将hash表写入分布式缓存中
   
   map任务:将数据从分布式缓存中读入内存，将表中数据逐一和hash表匹配。将匹配到数据进行合并并写入hdfs中
   无reduce任务
   ```

#### 执行层面

  1. 推测执行

     ```
     1. 机器中的负载是不一样的。 
     2. 集群中的配置是不一样的。
     3. 数据倾斜
     
       推测执行(Speculative Execution)是指在集群环境下运行MapReduce，可能是程序Bug，负载不均或者其他的一些问题，导致在一个JOB下的多个TASK速度不一致，比如有的任务已经完成，但是有些任务可能只跑了10%，根据木桶原理，这些任务将成为整个JOB的短板，如果集群启动了推测执行，这时为了最大限度的提高短板，Hadoop会为该task启动备份任务，让speculative task与原始task同时处理一份数据，哪个先运行完，则将谁的结果作为最终结果，并且在运行完成后Kill掉另外一个任务。
     ```

   2. 并行执行

      ```
      set hive.exec.parallel=true;   //打开任务并行执行
      set hive.exec.parallel.thread.number=16; //同一个sql允许最大并行度，默认为8。
      ```

   3. jvm重用

      ```
        JVM重用是hadoop调优参数的内容，对hive的性能具有非常大的影响，特别是对于很难避免小文件的场景或者task特别多的场景，这类场景大多数执行时间都很短。hadoop默认配置是使用派生JVM来执行map和reduce任务的，这是jvm的启动过程可能会造成相当大的开销，尤其是执行的job包含有成千上万个task任务的情况。
      
        JVM重用可以使得JVM实例在同一个JOB中重新使用N次，N的值可以在Hadoop的mapre-site.xml文件中进行设置
      set  mapred.job.reuse.jvm.num.tasks=10;
      
         JVM的一个缺点是，开启JVM重用将会一直占用使用到的task插槽，以便进行重用，直到任务完成后才能释放。如果某个“不平衡“的job中有几个reduce task 执行的时间要比其他reduce task消耗的时间多得多的话，那么保留的插槽就会一直空闲着却无法被其他的job使用，直到所有的task都结束了才会释放。
      ```

      