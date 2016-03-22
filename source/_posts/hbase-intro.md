title: HBase介绍
date: 2015-08-07 00:22:33
tags:
- hbase
- big data
categories:
- hbase
description: HBase在公司已经用过一段时间，在Flume中添加一个HBase sink将一些数据存储到HBase里 ...

---------------

## 前言 ##

HBase在公司已经用过一段时间，在Flume中添加一个HBase sink将一些数据存储到HBase里。

当时HBase也没学，看了看几个例子，了解了它是基于列的表设计之后，就马上上手了，而且也把东西做出来了。 现在记录一下HBase的一些学习笔记。


## HBase简介 ##

HBase是什么？

HBase是运行在hadoop上的数据库，是一个分布式的，扩展性高的，存储大数据的数据库。

HBase也是开源的，非关系型数据库。基于Google的Bigtable设计。

什么时候需要使用HBase？

需要实时地读写大数据。HBase的目的就是管理亿级的数据。


## HBase基本概念 ##

HBase是基于列设计的，那什么是基于列呢？

首先看下关系型数据库的表结构，第一行是table的所有列，第二行开始就是各个列对应的值：

| id | name | age | birth_date |
|:----:|:----:|:----:|:----:|
|  1   |  format1   |  11   |  1980-01-01   |
|  2   |  format2   |  22   |  1985-01-01   |
|  3   |  format3   |  33   |  1990-01-01   |

HBase的表结构是这样的：

| Row Key | Time Stamp | ColumnFamily contents | ColumnFamily names |
|:----:|:----:|:----:|:----:|
|  'me.format.hbase'   |  t1   |  contents:format = "format1"   |     |
|  'me.format.hbase'   |  t2   |  contents:title = "title1"  |     |
|  'me.format.hbase'   |  t3   |     | names:gogogo = "data1"  |

从上面这个HBase表的例子来说明HBase的存储结构。

Row Key：行的键值，其实就相当于这一行的标识符。上面的数据其实只有1行，因为他们的标识符是一样的。

TimeStamp：时间戳，创建数据的时间戳，hbase默认会自动生成

ColumnFamily：列的前缀，一列可以存储多条数据，具体存储什么类型的数据还需要另外一个标示符qualify，上面那个例子中，contents和names就是两个Column Family

ColumnFamily qualify：列前缀后的标识符，一个ColumnFamily可以有多个qualify。上面那个例子中format和title就是contents这个ColumnFamily的qualify。gogogo是names这个ColumnFamily的qualify

## HBase的启动 ##

HBase下载完之后解压，解压后使用以下命令启动hbase：

	$ ./bin/start-hbase.sh
    
启动之前注意，机器要装好jdk，并且启动hadoop。因为hbase底层数据是存储在hdfs上的。

## HBase的基本操作 ##

### 表的创建 ###

创建一个表名位tableName，ColumnFamily有contents和names的表，qualify不需要声明，每次添加数据随意指定qualify即可：

	create 'tableName',['contents', 'names']
    
### 表的删除 ###

删除表的所有数据：

	truncate table tableName

删除表，删除之前需要先disable表，然后才可删除：

	disable 'tableName'
	drop 'tableName'

### 数据查询 ###

查询tableName表数据：

	scan tableName
    
返回：

	ROW                                         COLUMN+CELL
     me.format.hbase                            column=contents:format, timestamp=1438875060466, value=format1

### 数据删除 ###

比如，表tableName里有如下数据：

    ROW                                         COLUMN+CELL
     me.format.hbase                            column=contents:format, timestamp=1438875566613, value=format1
     me.format.hbase                            column=contents:title, timestamp=1438875577687, value=title1
     me.format.hbase                            column=names:gogogo, timestamp=1438875597592, value=data1

进行删除操作，删除ColumnFamily qulify为contents:format的数据：

	delete 'tableName', 'me.format.hbase', 'contents:format'
    


### 数据修改 ###

HBase没有直接的update操作，只有put操作，put操作如果对应的地方有值，会覆盖：

	put 'tableName', 'me.format.hbase', 'contents:format', 'format111'
	

### 添加数据 ###

在tableName表里插入一个Row Key为me.format.hbase, ColumnFamily为contents，qualify为format，值的format1的数据：

	put 'tableName', 'me.format.hbase', 'contents:format', 'format1'
    
### 计数器 ###

HBase提供了一种计数器的概念，每次可以对某个值进行incr操作：

	incr 'tableName', 'me.format.hbase', 'contents:num', 1

查询数据：

    me.format.hbase                            column=contents:num, timestamp=1438875837362, value=\x00\x00\x00\x00\x00\x00\x00\x01
    
可以使用get_counter命令获得计数器的值：

	get_counter 'tableName','me.format.hbase', 'contents:num', 0
    
返回：

	COUNTER VALUE = 1
    
再次修改：

	incr 'tableName', 'me.format.hbase', 'contents:num', 100
    
    get_counter 'tableName','me.format.hbase', 'contents:num', 0
    
    COUNTER VALUE = 101
    
    incr 'tableName', 'me.format.hbase', 'contents:num', -102
    
    get_counter 'tableName','me.format.hbase', 'contents:num', 0
    
    COUNTER VALUE = -1
    
### 带条件的数据查询 ###

scan查询可以带几个参数。

COLUMNS： ColumnFamily和qualify的值
LIMIT：展示的个数
FILTER：过滤条件

比如有以下数据：

	ROW                                         COLUMN+CELL
     me.format.hbase                            column=contents:format, timestamp=1438875707700, value=format1
     me.format.hbase                            column=contents:num, timestamp=1438876106259, value=\xFF\xFF\xFF\xFF\xFF\xFF\xFF\xFF
     me.format.hbase                            column=contents:title, timestamp=1438875577687, value=title1
     me.format.hbase                            column=names:gogogo, timestamp=1438875597592, value=data1
     me.format.hbase1                           column=contents:format, timestamp=1438877417358, value=format1
     me.format.hbase2                           column=contents:format, timestamp=1438877422756, value=format1
     me.format.hbase3                           column=contents:format, timestamp=1438877427312, value=format1

查询ColumnFamily，qualify为contents:format的数据：

	scan 'tableName', { COLUMNS => "contents:format", LIMIT => 10 }
    
结果：

	ROW                                         COLUMN+CELL
     me.format.hbase                            column=contents:format, timestamp=1438875707700, value=format1
     me.format.hbase1                           column=contents:format, timestamp=1438877417358, value=format1
     me.format.hbase2                           column=contents:format, timestamp=1438877422756, value=format1
     me.format.hbase3                           column=contents:format, timestamp=1438877427312, value=format1
    
查询ColumnFamily未contents的数据：
    
    scan 'tableName', { COLUMNS => "contents", LIMIT => 10 }
    
结果：

    ROW                                         COLUMN+CELL
     me.format.hbase                            column=contents:format, timestamp=1438875707700, value=format1
     me.format.hbase                            column=contents:num, timestamp=1438876106259, value=\xFF\xFF\xFF\xFF\xFF\xFF\xFF\xFF
     me.format.hbase                            column=contents:title, timestamp=1438875577687, value=title1
     me.format.hbase1                           column=contents:format, timestamp=1438877417358, value=format1
     me.format.hbase2                           column=contents:format, timestamp=1438877422756, value=format1
     me.format.hbase3                           column=contents:format, timestamp=1438877427312, value=format1

查询ColumnFamily未contents的数据，并只展示2行数据：

	scan 'tableName', { COLUMNS => "contents", LIMIT => 2 }
    
结果：
    
    ROW                                         COLUMN+CELL
     me.format.hbase                            column=contents:format, timestamp=1438875707700, value=format1
     me.format.hbase                            column=contents:num, timestamp=1438876106259, value=\xFF\xFF\xFF\xFF\xFF\xFF\xFF\xFF
     me.format.hbase                            column=contents:title, timestamp=1438875577687, value=title1
     me.format.hbase1                           column=contents:format, timestamp=1438877417358, value=format1
     
     
查询ColumnFamily未contents的数据，并只展示2行数据：

	scan 'tableName', { COLUMNS => "contents", FILTER => "ValueFilter( =, 'binaryprefix:title' )" }
     
结果：

    ROW                                         COLUMN+CELL
     me.format.hbase                            column=contents:title, timestamp=1438875577687, value=title1
     

scan命令具体其他的参数就不一一列举了，可查询文档解决。
     
     
