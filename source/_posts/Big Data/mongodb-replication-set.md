---
title: MongoDB Replication Set 安装与配置
date: 2016-08-30 13:38:55
categories: Big Data
tags: 
 - MongoDB
 - Distribute Computing
---
本篇文章记录了MongoDB复制集集群配置过程及注意事项，并提供了相应配置说明，另外也对MongoDB复制集集群如何配置用户名密码登陆做了说明。本文采用系统为Ubuntu 14.04 x64版本，使用配置APT源的方式获取并安装最新版MongoDB。



## MongoDB APT源安装



根据官网说明配置MongoDB APT源并安装，相关链接[ install-mongodb-on-ubuntu ](http://docs.mongodb.org/manual/tutorial/install-mongodb-on-ubuntu/)。



### 获取并导入MongoDB public GPG Key



在线获取：


```bash
	sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 7F0CEB10
```


离线导入：


```bash
	wget -O - https://docs.mongodb.org/10gen-gpg-key.asc | sudo apt-key add -
```


### 添加MongoDB源



添加list：


```bash
	echo "deb http://repo.mongodb.org/apt/ubuntu "$(lsb_release -sc)"/mongodb-org/3.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-3.0.list
```


更新源：


```bash
	sudo apt-get update
```


### 安装最新版MongoDB


```bash
	sudo apt-get install -y mongodb-org
```


## 配置MongoDB复制集集群



### 修改配置文件实现复制集

  

使用APT源安装MongoDB后，其默认配置文件为/etc/mongod.conf，修改参数使其支持复制集，最终配置文件如下：


```bash
	# mongod.conf

	dbpath=/var/lib/mongodb    #数据库存储路径

	logpath=/var/log/mongodb/mongod.log    #Log文件存储路径

	logappend=true

	bind_ip=0.0.0.0    #绑定地址改为0.0.0.0使其能从外部访问

	replSet=replset-name    #复制集名称，同一复制集名称必须相同

	oplogSize=2048
```


这样配置好每个节点的MongoDB配置文件后启动每个节点的MongoDB：


```bash
	sudo service mongod start
```


然后通过MongoDB的命令行访问数据库：


```bash
	mongo
```


就能看见复制集合正常启动：



主节点：


```
	replset-name:PRIMARY>
```


子节点：


```
	replset-name:SECONDARY>
```


以上MongoDB复制集集群配置完成。



### 配置MongoDB复制集集群密码登陆



在无密码情况下登陆MongoDB复制集集群主节点admin数据库：


```bash
	mongo mongo_replset_primary_ip/admin
```


添加Root权限用户（管理集群）和Admin权限用户（管理集群数据）：


```javascript
	replset-name:PRIMARY> db.createUser({user: 'root', pwd: 'password', roles: [{role: 'root', db: 'admin'}]})

	replset-name:PRIMARY> db.createUser({user: 'admin', pwd: 'password', roles: [{role: 'userAdminAnyDatabase', db: 'admin'}]})
```


添加完成之后在某个节点上生成keyFile文件：


```bash
	openssl rand -base64 741 > mongodb-keyfile
```


将其拷贝到每个节点的某个相同目录下，如/etc/mongodb-keyfile，由于MongoDB启动时默认使用的是mongodb用户，因此将该文件拥有者改为mongodb，并将其权限改为600：


```bash
	sudo chown mongodb /etc/mongodb-keyfile

	sudo chmod 600 /etc/mongodb-keyfile
```


接下来在每个节点的/etc/mongod.conf中加入：


```bash
	keyFile = /var/lib/mongodb/mongodb-keyfile
```


重启每个节点的MongoDB:


```bash
	sudo service mongod restart
```


此时登陆每个节点MongoDB后，需要密码验证通过后才能进行操作，集群密码登陆配置完成。



### 调整MongoDB复制集配置实现PRIMARY节点相对固定化



由于MongoDB复制集中只有PRIMARY节点才能进行写操作，而不对集群默认配置进行修改每次启动时集群PRIMARY节点分配是随机的，为了使其相对固定化，需要调整每个节点在集群中的权重。



连接主节点admin数据库：


```bash
	mongo mongo_replset_primary_ip/admin
```


调整每个节点权重：


```javascript
	replset-name:PRIMARY> db.auth('root', 'password')

	replset-name:PRIMARY> cnf = rs.conf()

	replset-name:PRIMARY> cnf.members[i].priority = priority_i

	replset-name:PRIMARY> rs.reconfig(cnf)
```


将需要调整为PRIMARY节点的priority值（小于1）调整为所有节点中最大的即可，调整完成后重启MongoDB集群后只要该节点正常就一直为PRIMARY节点。

