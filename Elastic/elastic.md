操作系统

```SHELL
#切换到root用户修改
vim /etc/security/limits.conf
 
# 在最后面追加下面内容
es hard nofile 65536
es soft nofile 65536

####
vi /etc/sysctl.conf 
vm.max_map_count=655360
sysctl -p

```



ES

```
bin/elasticsearch -Ecluster.name=my_cluster -Epath.data=/esdata/my_cluster_node1 -Enode.name=node1 -Ehttp.port=9200 -d

```



metricbeat

```shell
# 创建模板
metricbeat setup --dashboards

# 启用模块
[root@registry metricbeat]# ./metricbeat modules enable mongodb

# 开启
metricbeat -e -c metricbeat.yml -d public
```



![](https://raw.githubusercontent.com/im-vincent/image/master/20191125111338.png)

