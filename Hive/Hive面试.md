# Hive

H-SQL如何转换为MapReduce

```
Hive将SQL转化为MapReduce的过程
1.Antlr定义SQL的语法规则，完成SQL词法，语法解析，将sql转化为抽象树AST TREE
2.遍历AST TREE，抽象出查询的基本组成单元QueryBlock
3.遍历QueryBlock，翻译为执行操作数OperatorTree
4.逻辑层优化器进行OperatorTree变换，合并不必要的ReduceSinkOperator，减少shuffle数据量
5.遍历OperatorTree,翻译为MapReduce任务
6.物理层优化器进行MapReduce任务的变换，生成最终的执行计划
```

如果定位一个H-SQL慢在哪里

```
HiveServer日志获取
Hive调优需要看HiveServer的运行日志及GC日志。
HiveServer日志路径为：HiveServer节点的/var/log/Bigdata/hive/hiveserver/。

MapReduce日志查看
Hive提交的MapReduce的日志中详细地记录了map和reduce任务的运行情况。

定位调优思路
第一步，分析SQL待处理的表及文件，统计待处理的表的文件数、数据量、条数、文件格式、压缩格式，分析是否有更适合的文件存储格式、压缩格式，是否能够使用上分区或分桶，是否有大量小文件需要map前合并。如果有优化空间，则执行优化。

第二步，分析SQL的结构，是否有重复的子查询可以存到中间表，是否可以使用相关性优化器，是否出现笛卡尔积需要去除，是否可以使用Multiple Insert语句，是否使用了低性能的UDF或SerDe需要替换。如果有优化空间，则执行优化。

第三步，分析SQL的操作，是否可利用MapJoin，是否可使用SMB Join，是否需要设置map端聚合，是否需要优化count(distinct)，是否全局排序可以使用局部排序代替，是否可以使用向量化优化、基于代价的优化等优化器。如果有优化空间，则执行优化。

第四步，观察SQL启动的MR运行情况，如果map运行缓慢，考虑减小Map处理的最大数据量提高并发度，考虑增大map的内存和虚拟核数；如果是reduce运行缓慢，是否有group by倾斜需要解决，是否有join倾斜需要处理，当大量重复数据做去重时减少Reduce数量，当大量匹配记录做关联时增加Reduce数量。
```

H-SQL与Spark-SQL的区别

H-SQL的调优方法

H-SQL如何定位与解决数据倾斜问题

```sql
什么是数据倾斜？
数据倾斜主要表现在，map/reduce程序执行时，reduce节点大部分执行完毕，但是有一个或者几个reduce节点运行很慢，导致整个程序的处理时间很长，这是因为某一个key的条数比其他key多很多(有时是百倍或者千倍之多)，这条Key所在的reduce节点所处理的数据量比其他节点就大很多，从而导致某几个节点迟迟运行不完。

常见容易出现数据倾斜的操作？
数据倾斜可能会发生在group过程和join过程。

1. 大表和小表关联时
比如，一个上千万行的记录表和一个几千行表之间join关联时，容易发生数据倾斜。为什么大表和小表容易发生数据倾斜(无非有的reduce执行时间被延迟)？
解决方式：
1) 多表关联时，将小表(关联键记录少的表)依次放到前面，这样可以触发reduce端更少的操作次数，减少运行时间。
2) 同时可以使用Map Join让小的维度表缓存到内存。在map端完成join过程，从而省略掉reduce端的工作。但是使用这个功能，需要开启map-side join的设置属性：set hive.auto.convert.join=true(默认是false)。同时还可以设置使用这个优化的小表的大小：set hive.mapjoin.smalltable.filesize=25000000(默认值25M)

2. 大表和大表的关联
比如：大表和大表关联，但是其中一张表的多是空值或者0比较多，容易shuffle给一个reduce，造成运行慢。
解决方式1：
select ...忽略......
from trackinfo a left outer join pm_info b
on ( case when (a.ext_field7 is not null and length(a.ext_field7) > 0 and a.ext_field7 rlike '^[0-9]+$')
then cast(a.ext_field7 as bigint)
else cast(ceiling(rand() * -65535) as bigint) end = b.id )
#将A表垃圾数据（为null,为0，以及其他类型的数据）赋一个随机的负数，然后将这些数据shuffle到不同reduce处理。

解决方式2：

2>当key值都是有效值时，解决办法为设置以下几个参数
set hive.exec.reducers.bytes.per.reducer = 1000000000
也就是每个节点的reduce 默认是处理1G大小的数据，如果你的join操作也产生了数据倾斜，那么你可以在hive中设定：

set hive.optimize.skewjoin = true;
set hive.skewjoin.key = skew_key_threshold （default = 100000）

hive在运行的时候没有办法判断哪个key会产生多大的倾斜，所以使用这个参数控制倾斜的阀值，如果超过这个值，新的值会发送给那些还没有达到的reduce，一般可以设置成你处理的总记录数/reduce个数的2-4倍都可以接受。

倾斜是经常会存在的，一般select的层数超过2层，翻译成执行计划多余3个以上的mapreduce job都会很容易产生倾斜，建议每次运行比较复杂的sql之前都可以设一下这个参数，如果你不知道设置多少，可以就按官方默认的1个reduce只处理1G的算法，那么 skew_key_threshold  = 1G/平均行长. 或者默认直接设成250000000 (差不多算平均行长4个字节)

3、其他情况数据倾斜的处理
1》比如因group by造成数据倾斜？
使用Hive对数据做一些类型统计的时候遇到过某种类型的数据量特别多，而其他类型数据的数据量特别少。当按照类型进行group by的时候，会将相同的group by字段的reduce任务需要的数据拉取到同一个节点进行聚合，而当其中每一组的数据量过大时，会出现其他组的计算已经完成而这里还没计算完成，其他节点的一直等待这个节点的任务执行完成，所以会看到一直map 100% reduce 99%的情况。

解决方式1：

hive.map.aggr=true (默认true)这个配置项代表是否在map端进行聚合，相当于Combiner。

hive.groupby.skewindata=true (默认false)

有数据倾斜的时候进行负载均衡，当选项设定为true，生成的查询计划会有两个MR Job。第一个MR Job中，Map的输出结果集合会随机分布到Reduce中，每个Reduce做部分聚合操作，并输出结果，这样处理的结果是相同的group by key有可能被分发到不同的Reduce中，从而达到负载均衡的目的；第二个MR Job再根据预处理的数据结果按照group by key分布到Reduce中(这个过程可以保证相同的group by key被分布到同一个Reduce中)，最后完成最终的聚合操作。

4、通用的一些数据倾斜的处理方法
1》Reduce个数太少
reduce数太少 set mapred.reduce.tasks=800;

默认是先设置hive.exec.reduces.bytes.per.reducer这个参数，设置了后Hive会自动计算reduce的个数，因此两个参数一般不同时使用。

2》当hiveQL中包含count(distinct )时
如果数据量非常大，执行如select a,count(distinct b) from t group by a;类型的SQL时，会出现数据倾斜的问题。

解决方式：使用sum...group by代替。如select a,sum(1) from (select a,b from t group by a,b) group by a;
```



根据给出H-SQL判断JOB个数并给出优化方案与优化后的JOB个数

