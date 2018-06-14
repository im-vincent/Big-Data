HDFS架构

![](http://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-hdfs/images/hdfsarchitecture.png)

1 Master(NameNode/NN)  带 N个Slaves(DataNode/DN)
HDFS/YARN/HBase

1个文件会被拆分成多个Block
blocksize：128M
130M ==> 2个Block： 128M 和 2M

NN：
1）负责客户端请求的响应
2）负责元数据（文件的名称、副本系数、Block存放的DN）的管理

DN：
1）存储用户的文件对应的数据块(Block)
2）要定期向NN发送心跳信息，汇报本身及其所有的block信息，健康状况

A typical deployment has a dedicated machine that runs only the NameNode software. 
Each of the other machines in the cluster runs one instance of the DataNode software.
The architecture does not preclude running multiple DataNodes on the same machine 
but in a real deployment that is rarely the case.

NameNode + N个DataNode
建议：NN和DN是部署在不同的节点上


replication factor：副本系数、副本因子

All blocks in a file except the last block are the same size



YARN架构：

![](http://hadoop.apache.org/docs/current/hadoop-yarn/hadoop-yarn-site/yarn_architecture.gif)

1）ResourceManager: RM
	整个集群同一时间提供服务的RM只有一个，负责集群资源的统一管理和调度
	处理客户端的请求： 提交一个作业、杀死一个作业
	监控我们的NM，一旦某个NM挂了，那么该NM上运行的任务需要告诉我们的AM来如何进行处理

2) NodeManager: NM
	整个集群中有多个，负责自己本身节点资源管理和使用
	定时向RM汇报本节点的资源使用情况
	接收并处理来自RM的各种命令：启动Container
	处理来自AM的命令
	单个节点的资源管理

3) ApplicationMaster: AM
	每个应用程序对应一个：MR、Spark，负责应用程序的管理
	为应用程序向RM申请资源（core、memory），分配给内部task
	需要与NM通信：启动/停止task，task是运行在container里面，AM也是运行在container里面

4) Container
	封装了CPU、Memory等资源的一个容器
	是一个任务运行环境的抽象

5) Client
	提交作业
	查询作业的运行进度
	杀死作业


