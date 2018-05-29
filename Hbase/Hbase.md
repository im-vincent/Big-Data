hadoop HA
----------

RM HA
----------

hbase
----------
	hmaster
	hregionserver


start-hbase.sh
----------------
	hbase-daemon.sh start master
	hbase-daemons.sh start regionserver


hbase的ha设置
----------------
	启动多个master即可。
	zk节点下:/hbase/backup-master

命令分组
-----------
	[general]
	version whoami
	
	[ddl]
	
	[namespace]类似于mysql的库概念.



insert into 
nosql : Not only SQL.
key-value
put



hbase shell操作
------------------
	$>hbase shell							//登录shell终端.
	$hbase>help								//
	$hbase>help	'list_namespace'			//查看特定的命令帮助
	$hbase>list_namespace					//列出名字空间(数据库)
	$hbase>list_namespace_tables 'default'	//列出名字空间(数据库)
	$hbase>create_namespace 'ns1'			//创建名字空间
	$hbase>help 'create'
	$hbase>create 'ns1:t1','f1'				//创建表,指定空间下
	$hbase>put 'ns1:t1','row1','f1:id',100		//插入数据
	$hbase>put 'ns1:t1','row1','f1:name','tom'	//
	
	$hbase>get 'ns1:t1','row1'					//查询指定row
	$hbase>scan 'ns1:t1'						//扫描表

通过编程API访问Hbase
-----------------------
	1.创建hbase模块
	2.添加依赖
		<?xml version="1.0" encoding="UTF-8"?>
		<project xmlns="http://maven.apache.org/POM/4.0.0"
				 xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
				 xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
			<modelVersion>4.0.0</modelVersion>
	
			<groupId>com.it18zhang</groupId>
			<artifactId>HbaseDemo</artifactId>
			<version>1.0-SNAPSHOT</version>
	
			<dependencies>
				<dependency>
					<groupId>org.apache.hbase</groupId>
					<artifactId>hbase-client</artifactId>
					<version>1.2.3</version>
				</dependency>
			</dependencies>
		</project>
	
	2'.复制hbase集群的hbase-site.xml文件到模块的src/main/resources目录下。


	3.编程实现
		package com.it18zhang.hbasedemo.test;
	
		import org.apache.hadoop.conf.Configuration;
		import org.apache.hadoop.hbase.HBaseConfiguration;
		import org.apache.hadoop.hbase.TableName;
		import org.apache.hadoop.hbase.client.*;
		import org.apache.hadoop.hbase.util.Bytes;
		import org.junit.Test;
	
		import java.io.IOException;
	
		/**
		 *
		 */
		public class TestCRUD {
			@Test
			public void put() throws Exception {
				//创建conf对象
				Configuration conf = HBaseConfiguration.create();
				//通过连接工厂创建连接对象
				Connection conn = ConnectionFactory.createConnection(conf);
				//通过连接查询tableName对象
				TableName tname = TableName.valueOf("ns1:t1");
				//获得table
				Table table = conn.getTable(tname);
	
				//通过bytes工具类创建字节数组(将字符串)
				byte[] rowid = Bytes.toBytes("row3");
	
				//创建put对象
				Put put = new Put(rowid);
	
				byte[] f1 = Bytes.toBytes("f1");
				byte[] id = Bytes.toBytes("id") ;
				byte[] value = Bytes.toBytes(102);
				put.addColumn(f1,id,value);
	
				//执行插入
				table.put(put);
			}
	
			@Test
			public void get() throws Exception {
				//创建conf对象
				Configuration conf = HBaseConfiguration.create();
				//通过连接工厂创建连接对象
				Connection conn = ConnectionFactory.createConnection(conf);
				//通过连接查询tableName对象
				TableName tname = TableName.valueOf("ns1:t1");
				//获得table
				Table table = conn.getTable(tname);
	
				//通过bytes工具类创建字节数组(将字符串)
				byte[] rowid = Bytes.toBytes("row3");
				Get get = new Get(Bytes.toBytes("row3"));
				Result r = table.get(get);
				byte[] idvalue = r.getValue(Bytes.toBytes("f1"),Bytes.toBytes("id"));
				System.out.println(Bytes.toInt(idvalue));
			}
		}
	
	4.stop-hbas.sh
		
	5.

hbase写入过程
---------------
	WAL			//write ahead log,写前日志。
	
	ta/hbase/meta/1588230740/info/da7a63e29d1c4fb588068ea9c187ae27

hbase基于hdfs
-------------------
	相同列族的数据存放在一个文件中。
	[表数据的存储目录结构构成]
	hdfs://s201:8020/hbase/data/${名字空间}/${表名}/${区域名称}/${列族名称}/${文件名}


	[WAL目录结构构成]
	hdfs://s201:8020/hbase/WALs/${区域服务器名称,主机名,端口号,时间戳}/


client端交互过程
--------------------
	0.hbase集群启动时，master负责分配区域到指定区域服务器。
	
	1.联系zk，找出meta表所在rs(regionserver)
		/hbase/meta-region-server
	
	2.定位row key,找到对应region server
		
	3.缓存信息在本地。
	
	4.联系RegionServer
	
	5.HRegionServer负责open HRegion对象，为每个列族创建Store对象，Store包含多个StoreFile实例，
	  他们是对HFile的轻量级封装。每个Store还对应了一个MemStore，用于内存存储数据。


百万数据批量插入
----------------------
	[100万:33921.]
	
	long start = System.currentTimeMillis() ;
	Configuration conf = HBaseConfiguration.create();
	Connection conn = ConnectionFactory.createConnection(conf);
	TableName tname = TableName.valueOf("ns1:t1");
	HTable table = (HTable)conn.getTable(tname);
	//不要自动清理缓冲区
	table.setAutoFlush(false);
	
	for(int i = 4 ; i < 1000000 ; i ++){
		Put put = new Put(Bytes.toBytes("row" + i)) ;
		//关闭写前日志
		put.setWriteToWAL(false);
		put.addColumn(Bytes.toBytes("f1"),Bytes.toBytes("id"),Bytes.toBytes(i));
		put.addColumn(Bytes.toBytes("f1"),Bytes.toBytes("name"),Bytes.toBytes("tom" + i));
		put.addColumn(Bytes.toBytes("f1"),Bytes.toBytes("age"),Bytes.toBytes(i % 100));
		table.put(put);
	
		if(i % 2000 == 0){
			table.flushCommits();
		}
	}
	//
	table.flushCommits();
	System.out.println(System.currentTimeMillis() - start );


hbase切割文件10G进行切割。
----------------------------
	<property>
		<name>hbase.hregion.max.filesize</name>
		<value>10737418240</value>
		<source>hbase-default.xml</source>
	</property>

hbase shell命令
-------------------------
	$hbase>flush 'ns1:t1'		//清理内存数据到磁盘。
	$hbase>count 'ns1:t1'		//统计函数
	$hbase>disable 'ns1:t1'		//删除表之前需要禁用表
	$hbase>drop 'ns1:t1'		//
	
	$hbase>scan 'hbase:meta'	//查看元数据表
	$hbase>split 'ns1:t1'		//切割表 
	$hbase>split ''				//切割区域

hbase和hadoop的ha集成
-----------------------
	1.在hbase-env.sh文件添加hadoop配置文件目录到HBASE_CLASSPATH环境变量并分发.
		[/soft/hbase/conf/hbase-env.sh]
		export HBASE_CLASSPATH=$HBASE_CLASSPATH:/soft/hadoop/etc/hadoop
	
	2.在hbase/conf/目录下创建到hadoop的hdfs-site.xml符号连接。
		$>ln -s /soft/hadoop/etc/hadoop/hdfs-site.xml /soft/hbase/conf/hdfs-site.xml
	
	3.修改hbase-site.xml文件中hbase.rootdir的目录值。
		[/soft/hbase/conf/hbase-site.xml]
	    <property>
	            <name>hbase.rootdir</name>
	            <value>hdfs://mycluster/hbase</value>
	    </property>
	
	4.将以上步骤分发处理。



拆分风暴
-------------------
	10G