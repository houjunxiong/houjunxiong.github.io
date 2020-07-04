---
title: Spark任务Yarn模式参数配置
date: 2018-12-14 13:38:55
categories: Big Data
tags: 
 - Spark SQL
 - Yarn
 - Distribute Computing
---

Spark任务Yarn模式启动需要配置Driver、Executor参数，以Spark SQL Thrift Server为例，其启动配置如下：

```bash
/usr/hdp/current/spark2-thriftserver/sbin/start-thriftserver.sh --master yarn-client --num-executors 10 --executor-cores=4 --conf spark.driver.memory=8g --executor-memory 2G --conf spark.yarn.executor.memoryOverhead=2048
```

