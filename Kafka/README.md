Kafka架构

![](https://kafka.apache.org/10/images/kafka-apis.png)

- producer 生产者，就是生产馒头
- consumer 消费者，就是吃馒头的
- broker 篮子
- topic 主题，给馒头打个标签，你只能吃你的，不能吃别人的。 



安装配置

```bash
# 解压
[hadoop@hadoop01 software]$ tar xvfz kafka_2.11-1.0.1.tgz -C ../app/

# 软连接
[hadoop@hadoop01 app]$ ln -s kafka_2.11-1.0.1/ kafka

# 增加环境变量
[hadoop@hadoop01 ~]$ vi .bash_profile 
export KAFKA_HOME=/home/hadoop/app/kafka
export PATH=$KAFKA_HOME/bin:$PATH

# 修改配置文件
[hadoop@hadoop01 ~]$ cd $KAFKA_HOME/config 
[hadoop@hadoop01 config]$ vi server.properties
# 日志位置
log.dirs=/home/hadoop/app/tmp/kafka-logs
# zk地址
zookeeper.connect=hadoop01:2181

# 启动kafka
[hadoop@hadoop01 kafka]$ bin/kafka-server-start.sh $KAFKA_HOME/config/server.properties
```

使用

```bash
# 创建topic
[hadoop@hadoop01 ~]$ kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic test
Created topic "test".

# 列出所有topic
[hadoop@hadoop01 ~]$ kafka-topics.sh --list --zookeeper localhost:2181
test

# 向topic中写入数据
[hadoop@hadoop01 ~]$ kafka-console-producer.sh --broker-list localhost:9092 --topic test

# 消费数据
[hadoop@hadoop01 ~]$ kafka-console-consumer.sh --zookeeper localhost:2181 --topic test --from-beginning

# 查看指定topic的详情
[hadoop@hadoop01 ~]$ kafka-topics.sh --describe --zookeeper localhost:2181 --topic test
Topic:test      PartitionCount:1        ReplicationFactor:1     Configs:
        Topic: test     Partition: 0    Leader: 0       Replicas: 0     Isr: 0
```

