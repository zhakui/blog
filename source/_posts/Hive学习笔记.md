---
title: Hive学习笔记
date: 2017-08-12 17:28:43
categories:
  BigData
tags: 
  - Hadoop
  - Hive
  - Server
  - Technology
---

# Hive简单介绍
　　Hive是一种以SQL风格进行任何大小数据分析的工具，其特点是通过 SQL处理Hadoop的大数据，数据规模可以伸缩扩展到100PB+，数据形式可以是结构或非结构数据。

# Hive与传统关系数据库比较
* 侧重于分析，而非实时在线交易
* 无事务机制
* 不像关系数据库那样可以随机进行 insert或update.
* 通过Hadoop的map/reduce进行分布式处理，传统数据库则没有
* 传统关系数据库只能拓展最多20个服务器，而Hive可以拓展到上百个服务器。

# 安装Hive
1. 首先我们需要安装Java环境和Hadoop环境
2. 从这里http://mirrors.shuosc.org/apache/hive/获取Hive安装包
3. 解压安装包到安装目录下
```
$ tar -xzvf apache-hive-2.2.0-bin.tar.gz
```
4. 添加环境变量
```
export HIVE_HOME={{/user/local/hive-2.2.0}}
export PATH=$HIVE_HOME/bin:$PATH
```
5. 配置hive-env.sh文件
```
HADOOP_HOME=/user/local/hadoop-2.8.1
export HIVE_CONF_DIR=/user/local/hive-2.2.0/conf
```
6. 在HDFS文件系统上创建数据仓库目录并赋予权限
```
  $ hdfs dfs -mkdir       /tmp
  $ hdfs dfs -mkdir       /user/hive/warehouse
  $ hdfs dfs -chmod g+w   /tmp
  $ hdfs dfs -chmod g+w   /user/hive/warehouse
```
7. 使用配置启动Hive，需要初始化Hive的metastore数据库
```
$ bin/schematool -initSchema -dbType derby #我在这里折腾了很久....
```
8. 如果报这个错误的话（message:Version information not found in metastore. ），需要修改下面这个配置项
```
<property>
    <name>hive.metastore.schema.verification</name>
    <value>true</value>  #这里把true修改成false
</property>
```
9. 到这一步就可以使用命令启动Hive了
```
$ bin/hive
```
# Hive数据类型

|数据类型|所占字节|开始支持版本|
|TINYINT|1byte，-128 ~ 127||	 
|SMALLINT|2byte，-32,768 ~ 32,767||	 
|INT|4byte,-2,147,483,648 ~ 2,147,483,647||	 
|BIGINT|8byte,-9,223,372,036,854,775,808 ~ 9,223,372,036,854,775,807||
|BOOLEAN||| 	 
|FLOAT|4byte单精度|| 
|DOUBLE|8byte双精度||
|STRING||| 
|BINARY||从Hive0.8.0开始支持|
|TIMESTAMP||从Hive0.8.0开始支持|
|DECIMAL||从Hive0.11.0开始支持|
|CHAR||从Hive0.13.0开始支持|
|VARCHAR||从Hive0.12.0开始支持|
|DATE||从Hive0.12.0开始支持|

# 数据库操作
## 创建数据库
```
CREATE (DATABASE|SCHEMA) [IF NOT EXISTS] database_name
  [COMMENT database_comment]
  [LOCATION hdfs_path]
  [WITH DBPROPERTIES (property_name=property_value, ...)];
```
## 删除数据库
```
DROP (DATABASE|SCHEMA) [IF EXISTS] database_name [RESTRICT|CASCADE];
```
## 修改数据库
```
ALTER (DATABASE|SCHEMA) database_name SET DBPROPERTIES (property_name=property_value, ...);   -- (Note: SCHEMA added in Hive 0.14.0)
 
ALTER (DATABASE|SCHEMA) database_name SET OWNER [USER|ROLE] user_or_role;   -- (Note: Hive 0.13.0 and later; SCHEMA added in Hive 0.14.0)
 
ALTER (DATABASE|SCHEMA) database_name SET LOCATION hdfs_path; -- (Note: Hive 2.2.1, 2.4.0 and later)
```
# 数据库表操作
## 创建表
```
CREATE [TEMPORARY] [EXTERNAL] TABLE [IF NOT EXISTS] [db_name.]table_name    -- (Note: TEMPORARY available in Hive 0.14.0 and later)
  [(col_name data_type [COMMENT col_comment], ... [constraint_specification])]
  [COMMENT table_comment]
  [PARTITIONED BY (col_name data_type [COMMENT col_comment], ...)]
  [CLUSTERED BY (col_name, col_name, ...) [SORTED BY (col_name [ASC|DESC], ...)] INTO num_buckets BUCKETS]
  [SKEWED BY (col_name, col_name, ...)                  -- (Note: Available in Hive 0.10.0 and later)]
     ON ((col_value, col_value, ...), (col_value, col_value, ...), ...)
     [STORED AS DIRECTORIES]
  [
   [ROW FORMAT row_format] #指定行分隔符
***[FIELDS TERMINATED BY ] #指定字段间的分隔符
   [STORED AS file_format]   
     | STORED BY 'storage.handler.class.name' [WITH SERDEPROPERTIES (...)]  -- (Note: Available in Hive 0.6.0 and later)
  ]
  [LOCATION hdfs_path]
  [TBLPROPERTIES (property_name=property_value, ...)]   -- (Note: Available in Hive 0.6.0 and later)
  [AS select_statement];   -- (Note: Available in Hive 0.5.0 and later; not supported for external tables)

CREATE [TEMPORARY] [EXTERNAL] TABLE [IF NOT EXISTS] [db_name.]table_name
  LIKE existing_table_or_view_name
  [LOCATION hdfs_path];
```
# 添加分区表
```
ALTER TABLE table_name ADD [IF NOT EXISTS] PARTITION partition_spec
[LOCATION 'location1'] partition_spec [LOCATION 'location2'] ...;

partition_spec:
: (p_column = p_col_value, p_column = p_col_value, ...)
```
把数据一定规则划分（例如：把收集的日志按天存放），把数据按天存储在一个单独的文件，可以减少了查询处理时间。

## 加载数据到表中
```
LOAD DATA [LOCAL] INPATH 'filepath' [OVERWRITE] INTO TABLE tablename 
[PARTITION (partcol1=val1, partcol2=val2 ...)]
```

# SQL语句
## 基本查询语句
```
SELECT [ALL | DISTINCT] select_expr, select_expr, ... 
FROM table_reference 
[WHERE where_condition] 
[GROUP BY col_list] 
[HAVING having_condition] 
[CLUSTER BY col_list |                   #col_list相同时，CLUSTER BY相当于是DISTRIBUTE BY和SORT BY的组合
 [DISTRIBUTE BY col_list]                #控制map的输出在reducer是如何划分的（默认是通过hash的方式划分）
 [SORT BY col_list]]                     #每个reducer端都会做排序
[LIMIT number];
```

## JOIN查询
```
join_table:

   table_reference JOIN table_factor [join_condition]
   | table_reference {LEFT|RIGHT|FULL} [OUTER] JOIN table_reference
   join_condition
   | table_reference LEFT SEMI JOIN table_reference join_condition
   | table_reference CROSS JOIN table_reference [join_condition]
```
