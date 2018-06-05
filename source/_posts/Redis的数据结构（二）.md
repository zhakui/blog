---
title: Redis的数据结构（二）
date: 2013-05-16 09:58:37
categories:
  Server 
tags: 
  - Rides
  - Technology
  - Server
---
---
在上文[Redis的数据结构(一)](http://zhkui.com/2017/05/15/Redis%E7%9A%84%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84/)中，我们了解了Redis的两种数据类型(String和Hash)，这篇文章我们将继续了解其他几种数据类型。在Redis中总共有5中数据类型：
- String——字符串
- Hash——字典
- List——列表
- Set——集合
- Sorted Set——有序集合

# 1. List——列表

一般意义上讲，列表就是有序元素的序列：10,20,1,2,3就是一个列表。但用数组实现的List和用Linked List实现的List，在属性方面大不相同。
Redis lists基于Linked Lists实现。这意味着即使在一个list中有数百万个元素，在头部或尾部添加一个元素的操作，其时间复杂度也是常数级别的。用LPUSH命令在十个元素的list头部添加新元素，和在千万元素list头部添加新元素的速度相同。那么，坏消息是什么？在数组实现的list中利用索引访问元素的速度极快，而同样的操作在linked list实现的list上没有那么快。
Redis Lists用linked list实现的原因是：对于数据库系统来说，至关重要的特性是：能非常快的在很大的列表上添加元素。另一个重要因素是，正如你将要看到的：Redis lists能在常数时间取得常数长度。

LPUSH 命令可向list的左边（头部）添加一个新元素，而RPUSH命令可向list的右边（尾部）添加一个新元素。最后LRANGE 命令可从list中取出一定范围的元素
```sh
$ redis-cli rpush messages "Hello"
OK
$ redis-cli rpush messages "Redis：hello"
OK
$ redis-cli rpush messages "I'm a NOSQL"
OK
$ redis-cli lrange messages 0 2
1. Hello
2. Redis：hello
3. I'm a NOSQL
```
注意LRANGE 带有两个索引，一定范围的第一个和最后一个元素。这两个索引都可以为负来告知Redis从尾部开始计数，因此-1表示最后一个元素，-2表示list中的倒数第二个元素，以此类推。
正如你可以从上面的例子中猜到的，list可被用来实现聊天系统。还可以作为不同进程间传递消息的队列。关键是，你可以每次都以原先添加的顺序访问数据。这不需要任何SQL ORDER BY 操作，将会非常快，也会很容易扩展到百万级别元素的规模。

在上面的例子里 ，我们将“对象”（此例中是简单消息）直接压入Redis list，但通常不应这么做，由于对象可能被多次引用：例如在一个list中维护其时间顺序，在一个集合中保存它的类别，只要有必要，它还会出现在其他list中，等等。
例如我们有一个新闻列表，我们将用户提交的链接（新闻）添加到list中，可以用下面的方法：
```sh
$ redis-cli incr next.news.id
(integer) 1
$ redis-cli set news:1:title "Redis的数据结构(一)"
OK
$ redis-cli set news:1:url "http://zhkui.com/2017/05/15/Redis%E7%9A%84%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84/"
OK
$ redis-cli lpush submitted.news 1
OK
```
我们自增一个key，很容易得到一个独一无二的自增ID，然后通过此ID创建对象–为对象的每个字段设置一个key。最后将新对象的ID压入submitted.news list。
在命令参考文档中还可以读到所有和list有关的命令。你可以删除元素，旋转list，根据索引获取和设置元素，当然也可以用LLEN得到list的长度。

# 2. Set--集合
Redis集合是未排序的集合，其元素是二进制安全的字符串。SADD命令可以向集合添加一个新元素。和sets相关的操作也有许多，比如检测某个元素是否存在，以及实现交集，并集，差集等等。
```·sh
$ redis-cli sadd set 1
(integer) 1
$ redis-cli sadd set 2
(integer) 1
$ redis-cli sadd set 3
(integer) 1
$ redis-cli smembers set
1. 3
2. 1
3. 2
```

# 3. Sorted Set--有序集合
和Sets相比，Sorted Sets是将 Set 中的元素增加了一个权重参数 score，使得集合中的元素能够按 score 进行有序排列，比如一个存储全班同学成绩的 Sorted Sets，其集合 value 可以是同学的学号，而 score 就可以是其考试得分，这样在数据插入集合的时候，就已经进行了天然的排序。另外还可以用 Sorted Sets 来做带权重的队列，比如普通消息的 score 为1，重要消息的 score 为2，然后工作线程可以选择按 score 的倒序来获取工作任务。让重要的任务优先执行。
1. 带有权重的元素，比如一个游戏的用户得分排行榜
2. 比较复杂的数据结构，一般用到的场景不算太多