---
title: 使用Docker搭建Geomesa集群
categories:
  - Big Data
tags:
  - Docker
  - GeoMesa
  - Distribute Computing
  - Spatial-Temporal Big Data
abbrlink: 7ad08436
date: 2016-08-30 13:33:28
---
本文记录了使用Docker搭建Geomesa集群的过程，该Docker基于Github项目[hadoop-cluster-docker](https://github.com/kiwenlau/hadoop-cluster-docker)，安装了Hadoop 2.7.2、Spark 2.0.0、Accumulo 1.7.2等分布式系统相关软件。

## 开发Docker镜像

```bash
	FROM ubuntu:14.04

    MAINTAINER EdwardHou <fansbasahou@gmail.com>

    WORKDIR /root

    # install openssh-server, openjdk and wget
    RUN apt-get update && apt-get install -y openssh-server wget vim unzip r-base gdal-bin

    # install jdk 8
    RUN wget -qO jdk8.tar.gz --header "Cookie: oraclelicense=accept-securebackup-cookie" http://download.oracle.com/otn-pub/java/jdk/8u101-b13/jdk-8u101-linux-x64.tar.gz && \
        tar -xzvf jdk8.tar.gz && \
        mkdir /usr/local/jvm && \
        mv jdk1.8.0_101 /usr/local/jvm/ && \
        rm jdk8.tar.gz && \
        update-alternatives --install /usr/bin/java java /usr/local/jvm/jdk1.8.0_101/bin/java 100 && \
        update-alternatives --install /usr/bin/javac javac /usr/local/jvm/jdk1.8.0_101/bin/javac 100

    # install hadoop 2.7.2
    RUN wget https://github.com/kiwenlau/compile-hadoop/releases/download/2.7.2/hadoop-2.7.2.tar.gz && \
        tar -xzvf hadoop-2.7.2.tar.gz && \
        mv hadoop-2.7.2 /usr/local/hadoop && \
        rm hadoop-2.7.2.tar.gz

    # install spark 2.0.0
    RUN wget http://www-us.apache.org/dist/spark/spark-2.0.0/spark-2.0.0-bin-hadoop2.7.tgz && \
        tar -zxvf spark-2.0.0-bin-hadoop2.7.tgz && \
        mv spark-2.0.0-bin-hadoop2.7 /usr/local/spark && \
        rm spark-2.0.0-bin-hadoop2.7.tgz

    # install zookeeper 3.4.8
    RUN wget http://www-eu.apache.org/dist/zookeeper/stable/zookeeper-3.4.8.tar.gz &&\
        tar -xzvf zookeeper-3.4.8.tar.gz &&\
        mv zookeeper-3.4.8 /usr/local/zookeeper && \
        rm zookeeper-3.4.8.tar.gz

    # install accumulo 1.7.2
    RUN wget http://mirrors.tuna.tsinghua.edu.cn/apache/accumulo/1.7.2/accumulo-1.7.2-bin.tar.gz && \
        tar -xzvf accumulo-1.7.2-bin.tar.gz && \
        mv accumulo-1.7.2 /usr/local/accumulo && \
        rm accumulo-1.7.2-bin.tar.gz

    # install geoserver 2.9.0
     RUN wget http://jaist.dl.sourceforge.net/project/geoserver/GeoServer/2.9.0/geoserver-2.9.0-bin.zip && \
        unzip geoserver-2.9.0-bin.zip && \
        rm geoserver-2.9.0-bin.zip && \
        mv geoserver-2.9.0 /usr/local/geoserver

    # install geomesa 1.2.5
    RUN wget http://repo.locationtech.org/content/repositories/geomesa-releases/org/locationtech/geomesa/geomesa-dist/1.2.5/geomesa-dist-1.2.5-bin.tar.gz && \
        tar -xzvf geomesa-dist-1.2.5-bin.tar.gz && \
        rm geomesa-dist-1.2.5-bin.tar.gz && \
        tar -xzvf geomesa-1.2.5/dist/tools/geomesa-tools-1.2.5-bin.tar.gz && \
        mv geomesa-tools-1.2.5 /usr/local/geomesa && \  
        cp geomesa-1.2.5/dist/accumulo/geomesa-accumulo-distributed-runtime-1.2.5.jar /root/ && \
        tar -xzvf geomesa-1.2.5/dist/gs-plugins/geomesa-accumulo-gs-plugin-1.2.5-install.tar.gz -C /usr/local/geoserver/webapps/geoserver/WEB-INF/lib/ && \
        rm -rf geomesa-1.2.5

    # set environment variable
    ENV JAVA_HOME=/usr/local/jvm/jdk1.8.0_101
    ENV HADOOP_HOME=/usr/local/hadoop
    ENV HADOOP_PREFIX=/usr/local/hadoop
    ENV HADOOP_CONF_DIR=/usr/local/hadoop/etc/hadoop
    ENV SPARK_HOME=/usr/local/spark
    ENV ZOOKEEPER_HOME=/usr/local/zookeeper
    ENV ACCUMULO_HOME=/usr/local/accumulo
    ENV GEOMESA_HOME=/usr/local/geomesa
    ENV GEOSERVER_HOME=/usr/local/geoserver
    ENV PATH=$PATH:/usr/local/jvm/jdk1.8.0_101/bin:/usr/local/hadoop/bin:/usr/local/hadoop/sbin:/usr/local/spark/bin:/usr/local/zookeeper/bin:/usr/local/accumulo/bin:/usr/local/geomesa/bin

    # ssh without key
    RUN ssh-keygen -t rsa -f ~/.ssh/id_rsa -P '' && \
        cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys

    RUN mkdir -p ~/hdfs/namenode && \ 
        mkdir -p ~/hdfs/datanode && \
        mkdir $HADOOP_HOME/logs &&\
        mkdir /root/zookeeper

    COPY config/* /tmp/
    COPY config/accumulo/* /usr/local/accumulo/conf/
    COPY entrypoint.sh /root/

    RUN cat /tmp/authorized_keys >> ~/.ssh/authorized_keys

    RUN cat /tmp/environment >> ~/.bashrc

    RUN mv /tmp/ssh_config ~/.ssh/config && \
        mv /tmp/hadoop-env.sh /usr/local/hadoop/etc/hadoop/hadoop-env.sh && \
        mv /tmp/hdfs-site.xml $HADOOP_HOME/etc/hadoop/hdfs-site.xml && \ 
        mv /tmp/core-site.xml $HADOOP_HOME/etc/hadoop/core-site.xml && \
        mv /tmp/mapred-site.xml $HADOOP_HOME/etc/hadoop/mapred-site.xml && \
        mv /tmp/yarn-site.xml $HADOOP_HOME/etc/hadoop/yarn-site.xml && \
        cp /tmp/slaves $HADOOP_HOME/etc/hadoop/slaves && \
        mv /tmp/start-all.sh ~/start-all.sh && \
        mv /tmp/stop-all.sh ~/stop-all.sh && \
        mv /tmp/init-all.sh ~/init-all.sh && \
        mv /tmp/run-wordcount.sh ~/run-wordcount.sh && \
        mv /tmp/slaves $SPARK_HOME/conf/slaves && \
        mv /tmp/spark-env.sh $SPARK_HOME/conf/spark-env.sh && \
        mv /tmp/zoo.cfg $ZOOKEEPER_HOME/conf/zoo.cfg && \
        mv /tmp/start.ini $GEOSERVER_HOME/start.ini && \
        cp /usr/local/accumulo/lib/htrace-core.jar /usr/local/geoserver/webapps/geoserver/WEB-INF/lib/ && \
        mv /tmp/install-hadoop-accumulo.sh /usr/local/geomesa/bin/install-hadoop-accumulo.sh

    RUN chmod +x ~/start-all.sh && \
        chmod +x ~/stop-all.sh && \
        chmod +x ~/init-all.sh && \
        chmod +x ~/run-wordcount.sh && \
        chmod +x $HADOOP_HOME/sbin/start-dfs.sh && \
        chmod +x $HADOOP_HOME/sbin/start-yarn.sh && \
        chmod +x $SPARK_HOME/sbin/start-all.sh && \
        chmod +x /root/entrypoint.sh

    # format namenode
    RUN /usr/local/hadoop/bin/hdfs namenode -format

    #VOLUMN /var/data

    ENTRYPOINT ["/root/entrypoint.sh"]

    CMD [ "sh", "-c", "service ssh start; /usr/local/zookeeper/bin/zkServer.sh start; bash"]
```

## 基于Docker构建集群

基于Docker构建Geomesa集群：

```bash
#!/bin/bash

docker network create --driver=bridge hadoop

# the default node number is 3
N=${1:-3}


# start hadoop master container
docker rm -f hadoop-master &> /dev/null
echo "start hadoop-master container..."
docker run -itd \
                --net=hadoop \
                -p 2188:22 \
        -p 50070:50070 \
                -p 8088:8088 \
                -p 8080:8080 \
                -p 7070:7070 \
        -p 50095:50095 \
                -p 8888:8888 \
        -p 7077:7077 \
        -p 9000:9000 \
        -v /Users/houjunxiong/Documents/Data/:/root/alldata \
                --name hadoop-master \
                --hostname hadoop-master \
                fansbasahou/geomesa:1.0 &> /dev/null


# start hadoop slave container
i=1
while [ $i -lt $N ]
do
    let j=i+1
    docker rm -f hadoop-slave$i &> /dev/null
    echo "start hadoop-slave$i container..."
    docker run -itd \
            -e MYID=$j \
                    --net=hadoop \
                    --name hadoop-slave$i \
                    --hostname hadoop-slave$i \
                    fansbasahou/geomesa:1.0 &> /dev/null
    i=$(( $i + 1 ))
done 

# get into hadoop master container
docker exec -it hadoop-master bash
```

进入集群Master节点，完成Accumulo初始化工作，并启动集群：

```bash
# 进入Hadoop Master节点
docker exec -it hadoop-master bash

# 初始化并启动集群
./init-all.sh
./start-all.sh
```

配置Geomesa：

```bash
# 拷贝相关JAR包进入HDFS
hdfs dfs -mkdir /classpath
hdfs dfs -put /root/geomesa-accumulo-distributed-runtime-1.2.5.jar /classpath

# 进入accumulo shell
accumulo shell -u root

# 配置geomesa数据表属性，namespace -> cybergis，user -> root， password -> root
> createnamespace cybergis
> grant NameSpace.CREATE_TABLE -ns cybergis -u root
> config -s general.vfs.context.classpath.cybergis=hdfs://hadoop-master:9000/classpath/geomesa-accumulo-distributed-runtime-1.2.5.jar
> config -ns myNamespace -s table.classpath.context=cybergis
```
