zookeeper安装

```bash
[hadoop@hadoop01 app]$ tar xvfz zookeeper-3.4.5-cdh5.14.0.tar.gz  -C  ../app/

[hadoop@hadoop01 app]$ ln -s zookeeper-3.4.5-cdh5.14.0 zookeeper

[hadoop@hadoop01 ~]$ vi .bash_profile
export ZOOKEEPER_HOME=/home/hadoop/app/zookeeper
export PATH=$ZOOKEEPER_HOME/bin:$PATH
```



配置：

```bash
# 添加一个zoo.cfg配置文件
[hadoop@hadoop01 ~]$ cd $ZOOKEEPER_HOME
[hadoop@hadoop01 conf]$ cp zoo_sample.cfg zoo.cfg

# 修改配置文件（zoo.cfg）
dataDir=/home/hadoop/app/tmp/zk

# 启动zk
[hadoop@hadoop01 zookeeper]$ bin/zkServer.sh start
```



客户端

```bash
[hadoop@hadoop01 ~]$ zkCli.sh 
Connecting to localhost:2181
2018-03-24 01:11:38,350 [myid:] - INFO  [main:Environment@100] - Client 
...
[zk: localhost:2181(CONNECTED) 0] ls / 
[zookeeper]
```

