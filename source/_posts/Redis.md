---
title: Rides介绍和安装
date: 2013-05-12 17:32:40
categories:
  server technology
tags: 
  Rides
---

## Redis介绍
Redis是一个开源，先进的key-value存储，并用于构建高性能，可扩展的Web应用程序的完美解决方案。

Redis从它的许多竞争继承来的三个主要特点：

* Redis数据库完全在内存中，使用磁盘仅用于持久性。

* 相比许多键值数据存储，Redis拥有一套较为丰富的数据类型。

* Redis可以将数据复制到任意数量的从服务器。

## Redis 优势
  * 异常快速：Redis的速度非常快，每秒能执行约11万集合，每秒约81000+条记录。

  * 支持丰富的数据类型：Redis支持最大多数开发人员已经知道像列表，集合，有序集合，散列数据类型。这使得它非常容易解决各种各样的问题，因为我们知道哪些问题是可以处理通过它的数据类型更好。

  * 操作都是原子性：所有Redis操作是原子的，这保证了如果两个客户端同时访问的Redis服务器将获得更新后的值。

  * 多功能实用工具：Redis是一个多实用的工具，可以在多个用例如缓存，消息，队列使用(Redis原生支持发布/订阅)，任何短暂的数据，应用程序，如Web应用程序会话，网页命中计数等。

## Redis安装

### 1.安装必要包
```sh
yum install gcc
```

### 2.下载
```sh
wget http://download.redis.io/releases/redis-3.0.0.tar.gz  
tar zxvf redis-3.0.0.tar.gz  
cd redis-3.0.0  
\#如果不加参数,linux下会报错  
make MALLOC=libc
```

### 3.启动
```sh
\#启动redis  
src/redis-server &  

\#关闭redis  
src/redis-cli shutdown</span>
```

### 4.测试  
```sh
$ src/redis-cli  
127.0.0.1:6379> set foo bar  
OK  
127.0.0.1:6379> get foo  
"bar"
```
  
### 5.redis中比较重要的工具  
* redis-server： Redis 服务器程序
* redis-cli： Redis命令行操作工具，也可以用telnet根据其纯文本协议
* redis-benchmark：Redis性能测试工具，测试Reids在你的系统及配置下的读写性能