---
title: Hadoop HA配置
date: 2017-08-11 17:28:43
categories:
  BigData
tags: 
  - Hadoop
  - Server
  - Technology
---

# 集群时间同步
如果集群节点时间不同步，可能会出现节点宕机或引发其它异常问题，所以在生产环境中一般通过配置NTP服务器实现集群时间同步。本集群在master节点设置ntp服务器
```
//切换root用户
$ su root
# yum install -y ntp
// 配置时间服务器
# vim /etc/ntp.conf
# 禁止所有机器连接ntp服务器
restrict default ignore
# 允许局域网内的所有机器连接ntp服务器
restrict 172.16.20.0 mask 255.255.255.0 nomodify notrap
# 使用本机作为时间服务器
server 127.127.1.0
// 启动ntp服务器
# service ntpd start
// 设置ntp服务器开机自动启动
# chkconfig ntpd on
```
集群其它节点通过执行crontab定时任务，每天在指定时间向ntp服务器进行时间同步
```
// 切换root用户
$ su root
// 执行定时任务，每天00:00向服务器同步时间，并写入日志
# crontab -e
00-59  * * * /usr/sbin/ntpdate hadoop-master1>> /home/hadoop/ntpd.log
// 查看任务
# crontab -l
```
# Zookeeper集群安装
Zookeeper是一个开源分布式协调服务,zookeeper服务可用于：统一命名服务、配置管理、锁服务、选举。大数据应用中主要使用Zookeeper的集群管理功能
首先下载zookeeper,我这里下载的是zookeeper-3.4.10，
```
wget https://mirrors.tuna.tsinghua.edu.cn/apache/zookeeper/zookeeper-3.4.10/zookeeper-3.4.10.tar.gz
//解压zookeeper
tar  -xvf  zookeeper-3.4.10.tar.gz
// 配置用户环境变量
$ vi  ~/.bashrc
export ZOOKEEPER_HOME=/root/zookeeper-3.4.10
export PATH=$PATH:$ZOOKEEPER_HOME/bin
// 使修改的环境变量生效
$ source  ~/.bashrc
// 修改zookeeper的配置文件
$ cd  /root/zookeeper-3.4.10/conf/
$ cp  zoo_sample.cfg  zoo.cfg
$ vi  zoo.cfg
# 客户端心跳时间(毫秒)
tickTime=2000
# 允许心跳间隔的最大时间
initLimit=10
# 同步时限
syncLimit=5
# 数据存储目录
dataDir=/root/zookeeper-3.4.10/data
# 数据日志存储目录
dataLogDir=/root/zookeeper-3.4.10/data/log
# 端口号
clientPort=2181
# 集群节点和服务端口配置
server.1=master:2888:3888
server.2=slave1:2888:3888
server.3=slave2:2888:3888
# 到每个节点的dataDir目录下创建一个myid文件，在文件里面写如对应服务名称：
例如节点master
server.1=master:2888:3888
myid文件里面写而入1
// 启动
$ zkServer.sh start
// 查看状态
$ zkServer.sh status
// 关闭
$ zkServer.sh stop
```
# Hadoop HA配置
## 配置core-site.xml文件
```
$ vi  core-site.xml

<configuration>
  <!-- 指定hdfs的nameservices名称为mycluster，与hdfs-site.xml的HA配置相同 -->
  <property>
    <name>fs.defaultFS</name>
    <value>hdfs://mycluster</value>
  </property>
	
  <!-- 指定缓存文件存储的路径 -->
  <property>
    <name>hadoop.tmp.dir</name>
    <value>/root/hadoop-2.8.1/tmp</value>
  </property>

 <!-- 配置hdfs文件被永久删除前保留的时间（单位：分钟），默认值为0表明垃圾回收站功能关闭 -->
  <property>
    <name>fs.trash.interval</name>
    <value>1440</value>
  </property>
  
  <!-- 指定zookeeper地址，配置HA时需要 -->
  <property>
    <name>ha.zookeeper.quorum</name>
    <value>master:2181,slave1:2181,slave2:2181</value>
  </property>
</configuration>
```
## 配置hdfs-site.xml文件
```
$ vi  hdfs-site.xml

<configuration>
  <!-- 指定hdfs元数据存储的路径 -->
  <property>
    <name>dfs.namenode.name.dir</name>
    <value>/root/hadoop-2.8.1/namenode</value>
  </property>

  <!-- 指定hdfs数据存储的路径 -->
  <property>
    <name>dfs.datanode.data.dir</name>
    <value>/root/hadoop-2.8.1/datanode</value>
  </property>
  
  <!-- 数据备份的个数 -->
  <property>
    <name>dfs.replication</name>
    <value>3</value>
  </property>

  <!-- 关闭权限验证 -->
  <property>
    <name>dfs.permissions.enabled</name>
    <value>false</value>
  </property>
  
  <!-- 开启WebHDFS功能（基于REST的接口服务） -->
  <property>
    <name>dfs.webhdfs.enabled</name>
    <value>true</value>
  </property>
  
  ### 以下为HDFS HA的配置
  <!-- 指定hdfs的nameservices名称为mycluster -->
  <property>
    <name>dfs.nameservices</name>
    <value>mycluster</value>
  </property>

  <!-- 指定mycluster的两个namenode的名称分别为nn1,nn2 -->
  <property>
    <name>dfs.ha.namenodes.mycluster</name>
    <value>nn1,nn2</value>
  </property>

  <!-- 配置nn1,nn2的rpc通信端口 -->
  <property>
    <name>dfs.namenode.rpc-address.mycluster.nn1</name>
    <value>master:8020</value>
  </property>
  <property>
    <name>dfs.namenode.rpc-address.mycluster.nn2</name>
    <value>slave1:8020</value>
  </property>

  <!-- 配置nn1,nn2的http通信端口 -->
  <property>
    <name>dfs.namenode.http-address.mycluster.nn1</name>
    <value>master:50070</value>
  </property>
  <property>
    <name>dfs.namenode.http-address.mycluster.nn2</name>
    <value>slave1:50070</value>
  </property>

  <!-- 指定namenode元数据存储在journalnode中的路径 -->
  <property>
    <name>dfs.namenode.shared.edits.dir</name>
<value>qjournal://master:8485;slave1:8485;slave2:8485/mycluste</value>
  </property>
  
  <!-- 指定journalnode日志文件存储的路径 -->
  <property>
    <name>dfs.journalnode.edits.dir</name>
    <value>/root/hadoop-2.8.1/journal</value>
  </property>

  <!-- 指定HDFS客户端连接active namenode的java类 -->
  <property>
<name>dfs.client.failover.proxy.provider.mycluster</name>
<value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value>
  </property>

  <!-- 配置隔离机制为ssh -->
  <property>
    <name>dfs.ha.fencing.methods</name>
    <value>sshfence</value>
  </property>

  <!-- 指定秘钥的位置 -->
  <property>
    <name>dfs.ha.fencing.ssh.private-key-files</name>
    <value>/root/.ssh/id_rsa</value>
  </property>
  
  <!-- 开启自动故障转移 -->
  <property>
    <name>dfs.ha.automatic-failover.enabled</name>
    <value>true</value>
  </property>
</configuration>
```
## 配置mapred-site.xml文件
```
$ vim mapred-site.xml

<configuration>
  <!-- 指定MapReduce计算框架使用YARN -->
  <property>
    <name>mapreduce.framework.name</name>
    <value>yarn</value>
  </property>

  <!-- 指定jobhistory server的rpc地址 -->
  <property>
    <name>mapreduce.jobhistory.address</name>
    <value>master:10020</value>
  </property>

  <!-- 指定jobhistory server的http地址 -->
  <property>
    <name>mapreduce.jobhistory.webapp.address</name>
    <value>master:19888</value>
  </property>

  <!-- 开启uber模式（针对小作业的优化） -->
  <property>
    <name>mapreduce.job.ubertask.enable</name>
    <value>true</value>
  </property>

  <!-- 配置启动uber模式的最大map数 -->
  <property>
    <name>mapreduce.job.ubertask.maxmaps</name>
    <value>9</value>
  </property>

  <!-- 配置启动uber模式的最大reduce数 -->
  <property>
    <name>mapreduce.job.ubertask.maxreduces</name>
    <value>1</value>
  </property>
</configuration>
```
## 配置yarn-site.xml文件
```
$ vim yarn-site.xml

<configuration>
  <!-- NodeManager上运行的附属服务，需配置成mapreduce_shuffle才可运行MapReduce程序 -->
  <property>
    <name>yarn.nodemanager.aux-services</name>
    <value>mapreduce_shuffle</value>
  </property>

  <!-- 配置Web Application Proxy安全代理（防止yarn被攻击） -->
  <property>
    <name>yarn.web-proxy.address</name>
    <value>master:8888</value>
  </property>
  
  <!-- 开启日志 -->
  <property>
    <name>yarn.log-aggregation-enable</name>
    <value>true</value>
  </property>

  <!-- 配置日志删除时间为7天，-1为禁用，单位为秒 -->
  <property>
    <name>yarn.log-aggregation.retain-seconds</name>
    <value>604800</value>
  </property>

  <!-- 修改日志目录 -->
  <property>
    <name>yarn.nodemanager.remote-app-log-dir</name>
    <value>/logs</value>
  </property>

  <!-- 配置nodemanager可用的资源内存 -->
  <property>
    <name>yarn.nodemanager.resource.memory-mb</name>
    <value>2048</value>
  </property>

  <!-- 配置nodemanager可用的资源CPU -->
  <property>
    <name>yarn.nodemanager.resource.cpu-vcores</name>
    <value>2</value>
  </property>

``` 

# YARN HA的配置
```
  <!-- 开启YARN HA -->
  <property>
    <name>yarn.resourcemanager.ha.enabled</name>
    <value>true</value>
  </property>

  <!-- 启用自动故障转移 -->
  <property>
    <name>yarn.resourcemanager.ha.automatic-failover.enabled</name>
    <value>true</value>
  </property>

  <!-- 指定YARN HA的名称 -->
  <property>
    <name>yarn.resourcemanager.cluster-id</name>
    <value>yarncluster</value>
  </property>

  <!-- 指定两个resourcemanager的名称 -->
  <property>
    <name>yarn.resourcemanager.ha.rm-ids</name>
    <value>rm1,rm2</value>
  </property>

  <!-- 配置rm1，rm2的主机 -->
  <property>
    <name>yarn.resourcemanager.hostname.rm1</name>
    <value>slave1</value>
  </property>
  <property>
    <name>yarn.resourcemanager.hostname.rm2</name>
    <value>slave2</value>
  </property>

  <!-- 配置YARN的http端口 -->
  <property>
    <name>yarn.resourcemanager.webapp.address.rm1</name>
    <value>slave1:8088</value>
  </property>	
  <property>
    <name>yarn.resourcemanager.webapp.address.rm2</name>
    <value>slave2:8088</value>
  </property>

  <!-- 配置zookeeper的地址 -->
  <property>
    <name>yarn.resourcemanager.zk-address</name>
    <value>master:2181,slave1:2181,slave2:2181</value>
  </property>

  <!-- 配置zookeeper的存储位置 -->
  <property>
    <name>yarn.resourcemanager.zk-state-store.parent-path</name>
    <value>/rmstore</value>
  </property>

  <!-- 开启yarn resourcemanager restart -->
  <property>
    <name>yarn.resourcemanager.recovery.enabled</name>
    <value>true</value>
  </property>

  <!-- 配置resourcemanager的状态存储到zookeeper中 -->
  <property>
    <name>yarn.resourcemanager.store.class</name>
    <value>org.apache.hadoop.yarn.server.resourcemanager.recovery.ZKRMStateStore</value>
  </property>

  <!-- 开启yarn nodemanager restart -->
  <property>
    <name>yarn.nodemanager.recovery.enabled</name>
    <value>true</value>
  </property>

  <!-- 配置nodemanager IPC的通信端口 -->
  <property>
    <name>yarn.nodemanager.address</name>
    <value>0.0.0.0:45454</value>
  </property>
</configuration>
```
# Hadoop集群的初始化
启动zookeeper集群（分别在master、slave1和slave2上执行）
```
$ zkServer.shstart
```
格式化ZKFC（在master上执行）
```
$ hdfs  zkfc  -formatZK
```
// 启动journalnode（分别在master、slave1和slave2上执行）
```
$ hadoop-daemon.shstart  journalnode
```
// 格式化HDFS（在master上执行）
```
$ hdfs namenode  -format
```
// 将格式化后master节点hadoop工作目录中的元数据目录复制到slave1节点
```
$scp-r /root/hadoop-2.8.1/namenode/*  slave1:/root/hadoop-2.8.1/namenode/
```
// 初始化完毕后可关闭journalnode（分别在slave1、slave2和slave3上执行）
```
$ hadoop-daemon.sh  stop  journalnode

# Hadoop集群的启动
启动zookeeper集群（分别在master、slave和slave2执行）
```
$ zkServer. shstart
```
启动HDFS（在master执行）
```
$ start-dfs.sh
```
启动YARN（在slave1执行）
```
$ start-yarn.sh
```
启动YARN的另一个ResourceManager（在master执行，用于容灾）
```
$ yarn-daemon.sh start resourcemanager

启动YARN的历史任务服务（在master1执行）
```
$ mr-jobhistory-daemon.sh starthistoryserver
```