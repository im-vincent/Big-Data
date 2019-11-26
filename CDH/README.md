# CDH6离线安装

CentOS7.7下进行CDH6集群的完全离线部署

## 文件下载

首先一些安装CDH6集群的必须文件要先在外网环境先下载好。

### Cloudera Manager 6.2.1

CM6 RPM：https://archive.cloudera.com/cm6/6.2.1/redhat7/yum/RPMS/x86_64/
需要下载该链接下的所有RPM文件，由于jdk1.8我在环境准备部分已经手动安装了，所以可以不用下载`RPMS/x86_64/`目录下的jdk包[oracle-j2sdk1.8-1.8.0+update181-1.x86_64.rpm](https://archive.cloudera.com/cm6/6.2.1/redhat7/yum/RPMS/x86_64/oracle-j2sdk1.8-1.8.0+update181-1.x86_64.rpm)，但是其他4个rpm包一定要下载，保存到`cloudera-repos`目录下。

ASC文件：https://archive.cloudera.com/cm6/6.2.1/allkeys.asc
同时还需要下载一个asc文件，同样保存到`cloudera-repos`目录下：

```shell
[root@localhost cloudera-repos]# tree 
.
├── cloudera-manager-agent-6.2.1-1426065.el7.x86_64.rpm
├── cloudera-manager-daemons-6.2.1-1426065.el7.x86_64.rpm
├── cloudera-manager-server-6.2.1-1426065.el7.x86_64.rpm
├── cloudera-manager-server-db-2-6.2.1-1426065.el7.x86_64.rpm
├── enterprise-debuginfo-6.2.1-1426065.el7.x86_64.rpm
└── oracle-j2sdk1.8-1.8.0+update181-1.x86_64.rpm
```

### MySQL JDBC驱动

要求使用5.1.26以上版本的jdbc驱动，可[点击这里](https://dev.mysql.com/get/Downloads/Connector-J/mysql-connector-java-5.1.47.tar.gz)直接下载`mysql-connector-java-5.1.47.tar.gz`

### CDH 6.2.1

CDH6 Parcels：https://archive.cloudera.com/cdh6/6.2.1/parcels/
需要下载[CDH-6.2.1-1.cdh6.2.1.p0.1425774-el7.parcel](https://archive.cloudera.com/cdh6/6.2.1/parcels/CDH-6.2.1-1.cdh6.2.1.p0.1425774-el7.parcel)和[manifest.json](https://archive.cloudera.com/cdh6/6.2.1/parcels/manifest.json)这两个文件

## 配置Cloudera Manager yum库

注意：不要尝试使用FTP搭建CM的YUM库！

首先安装`httpd`和`createrepo`：
`yum -y install httpd createrepo`
启动`httpd`服务并设置开机自启动：
`systemctl start httpd`
`systemctl enable httpd`
然后进入到前面准备好的存放Cloudera Manager RPM包的目录`cloudera-repos`下：
`cd /upload/cloudera-repos/`
生成RPM元数据：

```shell
[root@localhost cloudera-repos]# createrepo .

Spawning worker 0 with 2 pkgs
Spawning worker 1 with 2 pkgs
Spawning worker 2 with 1 pkgs
Spawning worker 3 with 1 pkgs
Workers Finished
Saving Primary metadata
Saving file lists metadata
Saving other metadata
Generating sqlite DBs
Sqlite DBs complete
```

然后将`cloudera-repos`目录移动到httpd的html目录下：
`mv cloudera-repos /var/www/html/`
确保可以通过浏览器查看到这些RPM包：

![](https://raw.githubusercontent.com/im-vincent/image/master/20191115161122.png)

接着在Cloudera Manager Server主机上创建cm6的repo文件（要把哪个节点作为Cloudera Manager Server节点，就在这个节点上创建repo文件）：

```
cd /etc/yum.repos.d
cat > cloudera-manager.repo <<EOF
[cloudera-manager]
name=Cloudera Manager 6.2.1
baseurl=http://10.117.130.146/cloudera-repos/
gpgcheck=0
enabled=1
EOF
```

保存，退出,然后执行`yum clean all && yum makecache`命令：

## 安装Cloudera Manager Server

这一步只需要在CM Server节点上操作。
执行下面的命令：
`yum install oracle-j2sdk1.8-1.8.0+update181-1 cloudera-manager-daemons cloudera-manager-agent cloudera-manager-server`
将会需要很多依赖包，所以说还是有必要搭一个局域网内yum源的：

Centos iso http://mirrors.zju.edu.cn/centos/7.7.1908/isos/x86_64/

`cd /var/www/html/centos7-1908-repo`
`mount -o loop /usr/local/src/CentOS-7-x86_64-Everything-1908.iso /var/www/html/centos7-1908-repo/`

```
cd /etc/yum.repos.d
cat > centos7-1908.repo <<EOF
[centos7-1908-repo]
name=centos7-1908-repo
baseurl=http://10.117.130.146/centos7-1908-repo/
gpgcheck=0
enabled=1
EOF
```



## 配置本地Parcel存储库

Cloudera Manager Server安装完成后，进入到本地Parcel存储库目录：
`cd /opt/cloudera/parcel-repo`
将第一部分下载的CDH Parcel文件（`CDH-6.0.1-1.cdh6.0.1.p0.590678-el7.parcel`和`manifest.json`）上传至该目录下，然后执行命令生成sha文件：
	`sha1sum CDH-6.2.1-1.cdh6.2.1.p0.1580995-el7.parcel | awk '{ print $1 }' > CDH-6.2.1-1.cdh6.2.1.p0.1580995-el7.parcel.sha`
然后执行下面的命令修改文件所有者：
`chown -R cloudera-scm:cloudera-scm /opt/cloudera/parcel-repo/*`
最终`/opt/cloudera/parcel-repo`目录内容如下：

## 安装数据库

MySQL的安装在环境准备部分中已经有说明，这里就跳过MySQL安装了。

### 数据库配置

CDH官方给的有一份推荐的MySQL的配置内容：

```
[mysqld]
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock
transaction-isolation = READ-COMMITTED
# Disabling symbolic-links is recommended to prevent assorted security risks;
# to do so, uncomment this line:
symbolic-links = 0
key_buffer_size = 32M
max_allowed_packet = 32M
thread_stack = 256K
thread_cache_size = 64
query_cache_limit = 8M
query_cache_size = 64M
query_cache_type = 1
max_connections = 550
#expire_logs_days = 10
#max_binlog_size = 100M
#log_bin should be on a disk with enough free space.
#Replace '/var/lib/mysql/mysql_binary_log' with an appropriate path for your
#system and chown the specified folder to the mysql user.
log_bin=/var/lib/mysql/mysql_binary_log
#In later versions of MySQL, if you enable the binary log and do not set
#a server_id, MySQL will not start. The server_id must be unique within
#the replicating group.
server_id=1
binlog_format = mixed
read_buffer_size = 2M
read_rnd_buffer_size = 16M
sort_buffer_size = 8M
join_buffer_size = 8M
# InnoDB settings
innodb_file_per_table = 1
innodb_flush_log_at_trx_commit = 2
innodb_log_buffer_size = 64M
innodb_buffer_pool_size = 4G
innodb_thread_concurrency = 8
innodb_flush_method = O_DIRECT
innodb_log_file_size = 512M
[mysqld_safe]
log-error=/var/log/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid

```

### 配置mysql jdbc驱动

从前面下载好的`mysql-connector-java-5.1.47.tar.gz`包中解压出`mysql-connector-java-5.1.47-bin.jar`文件，将`mysql-connector-java-5.1.47-bin.jar`文件上传至CM Server节点上的`/usr/share/java/`目录下并重命名为`mysql-connector-java.jar`（如果`/usr/share/java/`目录不存在，需要手动创建）：

```shell
wget https://cdn.mysql.com//Downloads/Connector-J/mysql-connector-java-5.1.47.tar.gz
tar zxvf mysql-connector-java-5.1.47.tar.gz
mkdir -p /usr/share/java/
cp mysql-connector-java-5.1.47/mysql-connector-java-5.1.47-bin.jar /usr/share/java/mysql-connector-java.jar
```

### 创建CDH所需要的数据库

根据所需要安装的服务参照下表创建对应的数据库以及数据库用户，数据库必须使用utf8编码，创建数据库时要记录好用户名及对应密码：

| 服务名                             | 数据库名 | 用户名 |
| ---------------------------------- | -------- | ------ |
| Cloudera Manager Server            | scm      | scm    |
| Activity Monitor                   | amon     | amon   |
| Reports Manager                    | rman     | rman   |
| Hue                                | hue      | hue    |
| Hive Metastore Server              | hive     | hive   |
| Sentry Server                      | sentry   | sentry |
| Cloudera Navigator Audit Server    | nav      | nav    |
| Cloudera Navigator Metadata Server | navms    | navms  |
| Oozie                              | oozie    | oozie  |

我这里就先创建4个数据库及对应用户：

```mysql
mysql> CREATE DATABASE scm DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
mysql> CREATE DATABASE amon DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
mysql> CREATE DATABASE rman DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
mysql> CREATE DATABASE hive DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
mysql> GRANT ALL ON scm.* TO 'scm'@'%' IDENTIFIED BY 'scm';
mysql> GRANT ALL ON amon.* TO 'amon'@'%' IDENTIFIED BY 'amon';
mysql> GRANT ALL ON rman.* TO 'rman'@'%' IDENTIFIED BY 'rman';
mysql> GRANT ALL ON hive.* TO 'hive'@'%' IDENTIFIED BY 'hive';
mysql> FLUSH PRIVILEGES;

MariaDB [mysql]> use mysql;
MariaDB [mysql]> grant all on *.* to 'temp'@'%' identified by 'temp' with grant option;

```

## 设置Cloudera Manager 数据库

Cloudera Manager Server包含一个配置数据库的脚本。

- mysql数据库与CM Server是同一台主机
  执行命令：`/opt/cloudera/cm/schema/scm_prepare_database.sh mysql scm scm`
- mysql数据库与CM Server不在同一台主机上
  执行命令：`/opt/cloudera/cm/schema/scm_prepare_database.sh mysql -h  --scm-host  scm scm`

## 安装CDH节点

### 启动Cloudera Manager Server服务

`systemctl start cloudera-scm-server`
然后等待Cloudera Manager Server启动，可能需要稍等一会儿，可以通过命令`tail -f /var/log/cloudera-scm-server/cloudera-scm-server.log`去监控服务启动状态。

当看到`INFO WebServerImpl:com.cloudera.server.cmf.WebServerImpl: Started Jetty server.`日志打印出来后，说明服务启动成功，可以通过浏览器访问Cloudera Manager WEB界面了。

### 访问Cloudera Manager WEB界面

打开浏览器，访问地址：`http://:7180`，默认账号和密码都为admin：



## 集群安装

![](https://raw.githubusercontent.com/im-vincent/image/master/20191118164316.png)

![](https://raw.githubusercontent.com/im-vincent/image/master/20191118164549.png)

```bash
[root@utility01 parcel-repo]# ll
total 2044276
-rwxrwxrwx. 1 cloudera-scm cloudera-scm 2093320258 Nov 18 16:57 CDH-6.2.1-1.cdh6.2.1.p0.1580995-el7.parcel
-rw-r--r--. 1 cloudera-scm cloudera-scm         41 Nov 18 17:00 CDH-6.2.1-1.cdh6.2.1.p0.1580995-el7.parcel.sha
-rw-r--r--. 1 cloudera-scm cloudera-scm      11327 Nov  4 19:28 manifest.json
[root@utility01 parcel-repo]# systemctl start cloudera-scm-server
```

### 增加lzo支持

![](https://raw.githubusercontent.com/im-vincent/image/master/20191118172534.png)

`https://archive.cloudera.com/gplextras6/6.2.1/parcels/`

![](https://raw.githubusercontent.com/im-vincent/image/master/20191118172736.png)

```
com.hadoop.compression.lzo.LzoCodec
com.hadoop.compression.lzo.LzopCodec
```

### 资源设置

MapReduce配置
在MapReduce Configuration选项卡上，您可以规划增加的特定于任务的内存容量。
第7步：MapReduce配置
对于CDH 5.5及更高版本，我们建议仅为map和reduce任务指定堆或容器大小。 未指定的值将根据设置mapreduce.job.heap.memory-mb.ratio计算。 此计算遵循Cloudera Manager并根据比率和容器大小计算堆大小。
步骤7A：MapReduce完整性检查
完整性检查MapReduce设置对容器的最小/最大属性。
通过步骤7A，您可以一目了然地确认所有最小和最大资源分配都在您设置的参数范围内。

连续调度
启用或禁用连续调度会更改YARN连续或基于节点心跳调度的频率。 对于较大的群集（超过75个节点）看到繁重的YARN工作负载，建议通常使用以下设置禁用连续调度：

   yarn.scheduler.fair.continuous-scheduling-enabled应为false
   yarn.scheduler.fair.assignmultiple应该是真的

在大型群集上，连续调度可能导致ResourceManager无响应，因为连续调度会遍历群集中的所有节点。

有关连续调度调优的更多信息，请参阅以下知识库文章：FairScheduler使用assignmultiple调整和连续调度

我们没有75个节点，设置为yarn.scheduler.fair.continuous-scheduling-enabled 为true
yarn.scheduler.fair.assignmultiple = true

小结：
yarn中容器内存的计算方法
yarn.nodemanager.resource.memory-mb
容器虚拟 CPU 内核
yarn.nodemanager.resource.cpu-vcores
//这2个值是更具最后用到的资源剩下来 多少给yarn的
提示:如果yarn.nodemanager.resource.memory-mb=2G yarn.nodemanager.resource.cpu-vcores=2 4个节点那么在192.168.0.142:8088/cluster 界面上就会有 “Memory Total=8G” “ VCores Total=8”

最小容器内存
yarn.scheduler.minimum-allocation-mb = 1G //推荐值
最小容器虚拟 CPU 内核数量
yarn.scheduler.minimum-allocation-vcores = 1 //推荐值
最大容器内存
yarn.scheduler.maximum-allocation-mb = < yarn.nodemanager.resource.memory-mb
最大容器虚拟 CPU 内核数量
yarn.scheduler.maximum-allocation-vcores = < yarn.nodemanager.resource.cpu-vcores //推荐值
容器内存增量
yarn.scheduler.increment-allocation-mb = 512M
容器虚拟 CPU 内核增量
yarn.scheduler.increment-allocation-vcores = 1 //推荐值

ApplicationMaster 虚拟 CPU 内核
yarn.app.mapreduce.am.resource.cpu-vcores = 1 //推荐值
ApplicationMaster 内存
yarn.app.mapreduce.am.resource.mb = 1024M
ApplicationMaster Java 选项库
yarn.app.mapreduce.am.command-opts = -Djava.net.preferIPv4Stack=true -Xmx768m

堆与容器大小之比
mapreduce.job.heap.memory-mb.ratio = 0.8 默认
Map 任务 CPU 虚拟内核
mapreduce.map.cpu.vcores = 1
Map 任务内存
mapreduce.map.memory.mb = 1024
Map 任务 Java 选项库
mapreduce.map.java.opts = 忽视 ignored
I/O 排序内存缓冲 (MiB)
mapreduce.task.io.sort.mb = 400
Reduce 任务 CPU 虚拟内核
mapreduce.reduce.cpu.vcores = 1
Reduce 任务内存
mapreduce.reduce.memory.mb = 1024M
Reduce 任务 Java 选项库
mapreduce.reduce.java.opts 忽视

Map 任务最大堆栈 819 //是 mapreduce.map.memory.mb *0.8
Reduce 任务最大堆栈 819M //mapreduce.reduce.memory.mb* 0.8
ApplicationMaster Java 最大堆栈 819.2 //这个数字是yarn.app.mapreduce.am.resource.mb * 0.8得到的
链接中，有建议的值，但是我们这是测试集群，资源很小，生产集群上具体见链接最后的建议值