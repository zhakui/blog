---
title: Redis的数据结构（一）
date: 2017-05-15 15:04:26
categories:
  server technology
tags: 
  Rides
---

  Redis是一种面向“键/值”对类型数据的分布式NoSQL数据库系统，特点是高性能，持久存储，适应高并发的应用场景。它起步较晚，发展迅速，目前已被许多大型机构采用，比如Github、Twitter、Weibo等。Redis并不是简单的key-value存储，实际上他是一个数据结构服务器，支持不同类型的值。也就是说，你不必仅仅把字符串当作键所指向的值。在Redis中一共有5种数据结构，那每种数据结构的使用场景都是什么呢？

- String——字符串
- Hash——字典
- List——列表
- Set——集合
- Sorted Set——有序集合

# 1. String--字符串

String 数据结构是简单的 key-value 类型，value 不仅可以是 String，也可以是数字（当数字类型用 Long 可以表示的时候encoding 就是整型，其他都存储在 sdshdr 当做字符串）。使用 Strings 类型，可以完全实现目前 Memcached 的功能，并且效率更高。还可以享受 Redis 的定时持久化（可以选择 RDB 模式或者 AOF 模式），操作日志及 Replication 等功能。

## #SET、GET操作
	$ redis-cli set mykey "my binary safe value"
	OK
	$ redis-cli get mykey
	my binary safe value

> GET和SET用来获取和设置字符串的值，只可以是任何类型的字符串，甚至你都可以在一个键下保存一副jpeg图片。但值的长度不能超过1GB。

## #INCR操作

	$ redis-cli set count 100
	OK $ redis-cli incr count
	(integer) 101
	$ redis-cli incr count
	(integer) 102
	$ redis-cli incrby count 10
	(integer) 112
	
INCR 命令将字符串值解析成整型，将其加一，最后将结果保存为新的字符串值，类似的命令有INCRBY, DECR and DECRBY。实际上他们在内部就是同一个命令，只是看上去有点儿不同。对字符串，另一个的令人感兴趣的操作是GETSET命令，行如其名：他为key设置新值并且返回原值。这有什么用处呢？例如：你的系统每当有新用户访问时就用INCR命令操作一个Redis key。你希望每小时对这个信息收集一次。你就可以GETSET这个key并给其赋值0并读取原值。除了提供与 Memcached 一样的 get、set、incr、decr 等操作外，Redis 还提供了下面一些操作：

1. LEN niushuai：O(1)获取字符串长度
2. APPEND niushuai redis：往字符串 append 内容，而且采用智能分配内存（每次2倍）
3. 设置和获取字符串的某一段内容
4. 设置及获取字符串的某一位（bit）
5. 批量设置一系列字符串的内容

# 2. Hash--字典

hash是一个string类型的field和value的映射表.一个key可对应多个field，一个field对应一个value。将一个对象存储为hash类型，较于每个字段都存储成string类型更能节省内存。新建一个hash对象时开始是用zipmap(又称为small hash)来存储的。这个zipmap其实并不是hash table，但是zipmap相比正常的hash实现可以节省不少hash本身需要的一些元数据存储开销。尽管zipmap的添加，删除，查找都是O(n)，但是由于一般对象的field数量都不太多。所以使用zipmap也是很快的,也就是说添加删除平均还是O(1)。如果field或者value的大小超出一定限制后，Redis会在内部自动将zipmap替换成正常的hash实现.

## #HSET
HSET key field value
将哈希表key中的域field的值设为value。如果key不存在，一个新的哈希表被创建并进行hset操作。如果域field已经存在于哈希表中，旧值将被覆盖。

## #HGET
HGET key field
返回哈希表key中指定的field的值。

## #HSETNX
HSETNX key field value
将哈希表key中的域field的值设置为value，当且仅当域field不存在。若域field已经存在，该操作无效。如果key不存在，一个新哈希表被创建并执行hsetnx命令。

## #HMSET
HMSET key field value [field value ...]
同时将多个field - value(域-值)对设置到哈希表key中。此命令会覆盖哈希表中已存在的域。如果key不存在，一个空哈希表被创建并执行hmset操作。

## #HMGET
HMGET key field [field ...]
返回哈希表key中，一个或多个给定域的值。如果给定的域不存在于哈希表，那么返回一个nil值。因为不存在的key被当作一个空哈希表来处理，所以对一个不存在的key进行hmget操作将返回一个只带有nil值的表。

## #HGETALL
HGETALL key
返回哈希表key中，所有的域和值。在返回值里，紧跟每个域名(field name)之后是域的值(value)，所以返回值的长度是哈希表大小的两倍。

## #HDEL
HDEL key field [field ...]
删除哈希表key中的一个或多个指定域，不存在的域将被忽略。

## #HLEN
HLEN key
返回哈希表key对应的field的数量。

## #HEXISTS
HEXISTS key field
查看哈希表key中，给定域field是否存在。

## #HKEYS
HKEYS key
获得哈希表中key对应的所有field。

## #HVALS
HVALS key
获得哈希表中key对应的所有values。

## #hincrby
为哈希表key中的域field的值加上增量increment。增量也可以为负数，相当于对给定域进行减法操作。如果key不存在，一个新的哈希表被创建并执行hincrby命令。如果域field不存在，那么在执行命令前，域的值被初始化为0。对一个储存字符串值的域field执行hincrby命令将造成一个错误。本操作的值限制在64位(bit)有符号数字表示之内。
