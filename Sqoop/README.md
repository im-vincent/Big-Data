环境：CDH5.7 



Make sure HIVE_CONF_DIR is set correctly

```bash
vi .bash_profile
export HADOOP_CLASSPATH=$HADOOP_CLASSPATH:/opt/cloudera/parcels/CDH/lib/hive/lib/*
```

```bash
cp /etc/hive/conf/hive-* /etc/sqoop/conf/
```



建议先在hive建立表结构。因为你不建立表结构，默认会把Oracle数据库里的int类型转成double类型。

基本导入命令

```bash
create table dw.dim_age (age_key int, level1_id STRING, level1_name STRING, level2_id STRING, level2_name STRING, begin_value int, end_value int, begin_date STRING, end_date STRING);

sqoop import \
--hive-import \
--connect jdbc:oracle:thin:@10.117.130.29:1521:cisda \
--username wangxi \
--password wangxi \
--table CISDA_GZ.DIM_AGE \
--hive-table dw.dim_age \
--fetch-size 2000 \
--null-string '\\N' \
--null-non-string '\\N' \
-m 1
```

ods

```SQL
create
	table
		ods.fact_costs_detail ( area_key int,
		medical_key int,
		age_key int,
		insur_key int,
		hospital_id string,
		disease_key string,
		patient_id string,
		hisid string,
		item_id_old string,
		item_id string,
		ptype string,
		frequency int,
		total_costs double,
		eligible_amount double,
		physician_name string,
		physician_id string,
		deptname string,
		dept_id string,
		reject_money double,
		detail_id string,
		rule_bit int,
         time_key string );
```

ods导入数据

```bash
sqoop import \
 	--hive-import \
 	--connect jdbc:oracle:thin:@10.117.130.29:1521:cisda \
 	--username wangxi \
 	--password wangxi \
	--hive-table ods.fact_costs_detail \
 	--target-dir /user/hive/warehouse/ods.db/temp_fact_costs_detail \
 	--fetch-size 2000 \
 	--split-by hisid \
 	--null-string '\\N' \
	--null-non-string '\\N' \
 	-m 2 \
 	--query "select area_key, medical_key, age_key, insur_key, hospital_id, disease_key, patient_id, hisid, item_id_old, item_id, ptype, frequency, total_costs, eligible_amount, physician_name, physician_id, deptname, dept_id, reject_money, detail_id, rule_bit, to_char(TIME_KEY,'yyyymmdd') as TIME_KEY from cisda_gz.fact_costs_detail t where \$CONDITIONS"
```

dw

```sql
create
	table
		dw.fact_costs_detail ( area_key string,
		medical_key string,
		age_key string,
		insur_key string,
		hospital_id string,
		disease_key string,
		patient_id string,
		hisid string,
		item_id_old string,
		item_id string,
		ptype string,
		frequency string,
		total_costs double,
		eligible_amount double,
		physician_name string,
		physician_id string,
		deptname string,
		dept_id string,
		reject_money double,
		detail_id string,
		rule_bit string ) partitioned by ( time_key string ) stored as parquet;
```

insert 

```sql
SET hive.exec.dynamic.partition=true;  
SET hive.exec.dynamic.partition.mode=nonstrict; 
SET hive.exec.max.dynamic.partitions.pernode = 1000;
SET hive.exec.max.dynamic.partitions=1000;
insert
	into
		dw.fact_costs_detail partition ( time_key ) select
			area_key,
			medical_key,
			age_key,
			insur_key,
			hospital_id,
			disease_key,
			patient_id,
			hisid,
			item_id_old,
			item_id,
			ptype,
			frequency,
			total_costs,
			eligible_amount,
			physician_name,
			physician_id,
			deptname,
			dept_id,
			reject_money,
			detail_id,
			rule_bit,
			from_unixtime( unix_timestamp( time_key, 'yyyyMMdd' ),
			'yyyyMM' )
		from
			ods.fact_costs_detail;
```

