---
title: 大数据TPC-H测试
date: 2018-12-14 13:38:55
categories: Big Data
tags: 
 - Spark SQL
 - Presto
 - Apache Kylin
 - TPC-H
---

## TPC-H标准

TPC Benchmark H（TPC-H）是一个决策支持的基准，由一系列面向上午应用的查询和并行数据修改组成。基准中选择的查询和组成数据库的数据在商业上都具有广泛的调表性并且易于实现。TPC-H阐明了决策支持系统三方面能力：

- 分析大量数据
- 执行高复杂度查询
- 回答关键的且经常需要回答的商业问题

本文基于TPC-H标准对基于HDFS的数据仓库查询框架Spark SQL、Presto以及Apache Kylin进行测试，以比较三者性能。

## 数据准备

本文采用[Hive-TestBench](https://github.com/hortonworks/hive-testbench)生成TPC-H常用数据集并存入Apache Hive数据仓库中，本次实验采用的数据格式为Apache ORC、数据大小是10GB，数据集事实表lineitem共包含5000万+条记录。Hive-TestBench的使用如下所示：

```bash
# 下载hive-testbench源码并进入目录
git clone https://github.com/hortonworks/hive-testbench.git
cd hive-testbench
# 编译hive-testbench TPC-H
./tpch-build.sh # 编译TPC-DS使用./tpcds-build.sh命令
# 生成10GB TPC-H测试数据
./tpch-setup.sh 10 # 生成TPC-DS使用./tpcds-set.sh 10，还可以使用FORMAT=PARQUET ./tpcds-set.sh 10限定生成数据格式为Parquet格式或者FORMAT=rcfile ./tpcds-set.sh 10限定生成数据格式为Apache ORC格式
```

## 环境准备

### 准备Spark SQL环境

本文计划测试Spark SQL Thrift Server性能，因此需要先行启动Spark SQL Thrift Server，启动命令如下：

```bash
start-thriftserver.sh --master yarn-client --num-executors 10 --executor-cores=4 --conf spark.driver.memory=8g --executor-memory 2G --conf spark.yarn.executor.memoryOverhead=2048
```

### 准备Presto环境

为保证测试结果有对比性，Presto所使用的计算资源应与Spark SQL Thrift Server一致，因此需要部署包括1个Coordinator节点、10个Worker节点（每节点的内存配置为2GB）的Presto集群。Presto Coordinator节点配置如下：

```properties
coordinator=true
node-scheduler.include-coordinator=false
http-server.http.port=18888
query.max-memory=50GB
query.max-memory-per-node=2GB
query.max-total-memory-per-node=4GB
discovery-server.enabled=true
discovery.uri=http://kmr-2389b0bb-gn-9fca5a4f-master-1-001.ksc.com:18888
```

Worker节点配置如下：

```properties
coordinator=false
http-server.http.port=18888
query.max-memory=50GB
query.max-memory-per-node=2GB
query.max-total-memory-per-node=4GB
discovery.uri=http://kmr-2389b0bb-gn-9fca5a4f-master-1-001.ksc.com:18888
```

同时，Presto需要连接Apache Hive数据源，配置如下：

```properties
connector.name=hive-hadoop2
hive.metastore.uri=thrift://kmr-2389b0bb-gn-9fca5a4f-master-1-001.ksc.com:9083
hive.config.resources=/etc/hadoop/conf/core-site.xml,/etc/hadoop/conf/hdfs-site.xml
```

**注意**：Presto同时Hive MetaStore Server连接Apache Hive数据源，而不是直接通过Thrift Server，所以需要预先启动Apache Hive的MetaStore服务；此外，Presto需要配置HDFS参数即core-site.xml及hdfs-site.xml两个文件。

Presto的自动化部署脚本如下：

```bash
#!/bin/sh
echo stop presto 0.214
`pwd`/presto_stop.sh
echo start deploy presto 0.214 to coordinator
rm -rf /usr/lib/presto-server-0.214
rm -rf /var/presto
tar -zxf presto-server-0.214.tar.gz -C /usr/lib/
mkdir -p /usr/lib/presto-server-0.214/etc
cp etc/node.properties /usr/lib/presto-server-0.214/etc/
cp etc/config.coordinator.properties /usr/lib/presto-server-0.214/etc/config.properties
cp etc/jvm.config /usr/lib/presto-server-0.214/etc/
cp -r etc/catalog /usr/lib/presto-server-0.214/etc/
sed -i s/node_id/`uuidgen`/g /usr/lib/presto-server-0.214/etc/node.properties
cp presto-cli-0.214-executable.jar /usr/lib/presto-server-0.214/bin/presto
for i in {1..10};
do
        host=`printf "kmr-2389b0bb-gn-9fca5a4f-core-1-%03d.ksc.com" $i`
        echo start deploy presto 0.214 to worker $host
        ssh root@$host 'rm -rf /usr/lib/presto-server-0.214'
        ssh root@$host 'rm -rf /var/presto'
        scp -r presto-server-0.214.tar.gz root@$host:/usr/lib/
        ssh root@$host 'tar -zxf /usr/lib/presto-server-0.214.tar.gz -C /usr/lib/'
        ssh root@$host 'mkdir -p /usr/lib/presto-server-0.214/etc'
        scp etc/node.properties root@$host:/usr/lib/presto-server-0.214/etc/
        scp etc/config.worker.properties root@$host:/usr/lib/presto-server-0.214/etc/config.properties
        scp etc/jvm.config root@$host:/usr/lib/presto-server-0.214/etc/
        scp -r etc/catalog root@$host:/usr/lib/presto-server-0.214/etc/
        ssh root@$host 'sed -i s/node_id/`uuidgen`/g /usr/lib/presto-server-0.214/etc/node.properties'
done
`pwd`/presto_start.sh
```

Presto的启动脚本如下：

```bash
#!/bin/sh
echo start presto 0.214 coordinator
/usr/lib/presto-server-0.214/bin/launcher start
for i in {1..10};
do
        host=`printf "kmr-2389b0bb-gn-9fca5a4f-core-1-%03d.ksc.com" $i`
        echo start presto 0.214 worker $host
        ssh root@$host '/usr/lib/presto-server-0.214/bin/launcher start'
done
```

Presto的停止脚本如下：

```bash
#!/bin/sh
echo stop presto 0.214 coordinator
/usr/lib/presto-server-0.214/bin/launcher stop
for i in {1..10};
do
        host=`printf "kmr-2389b0bb-gn-9fca5a4f-core-1-%03d.ksc.com" $i`
        echo stop presto 0.214 worker $host
        ssh root@$host '/usr/lib/presto-server-0.214/bin/launcher stop'
done
```

### 准备Apache Kylin环境

Apache Kylin的安装请见[Apache Kylin官方文档](http://kylin.apache.org/cn/docs/)。在进行Apache Kylin TPC-H测试前，需要对TPC-H测试数据构建CUBE，以保证Apache Kylin能够对其进行快速查询，CUBE建立参见Github [kylin-tpch](https://github.com/Kyligence/kylin-tpch)项目，配置构建CUBE过程如下：

```bash
# 下载kylin-tpch源码并进入目录
git clone https://github.com/Kyligence/kylin-tpch.git
cd kylin-tpch
# 构建Model
./setup-kylin-model.sh 10 # 后面的数值型参数与前序hive-testbench生成数据时参数一致
# 进入Apache Kylin前端全量构建CUBE：lineitem_cube、partsupp_cube、customer_cube、customer_vorder_cube
```

## TPC-H测试

### Spark SQL Thrift Server测试

采用beeline方式连接Spark SQL Thrift Server进行测试，其命令如下：

```bash
beeline -u jdbc:hive2://$host:$port/$database -n $username -p $password -f $sql_path
```

### Presto测试

采用Presto命令行进行测试，其命令如下：

```bash
presto --server $host:$port --catalog hive --schema $database -f $sql_path
```

### Kylin测试

采用kylin-tpch进行测试，其命令如下：

```bash
# 进入kylin-tpch目录
cd kylin-tpch
# 调用python脚本进行测试
python tools/query_tool.py -s http://127.0.0.1:7070/kylin -u ADMIN -p KYLIN -d queries -o tpch -r 3 -t kylin
```

### 测试结果

![image-20181214163213593](https://cdn.jsdelivr.net/gh/houjunxiong/houjunxiong@images/uPic/image-20181214163213593-1593866796634.png)

