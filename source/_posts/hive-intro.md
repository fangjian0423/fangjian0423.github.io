title: Hive介绍
date: 2015-07-31 13:32:33
tags:
- hive
- big data
categories: 
- hive
description: Hive是基于Hadoop的一个数据仓库工具，使用它可以查询和管理分布式存储系统上的大数据集 ...

---------------

Hive是基于Hadoop的一个数据仓库工具，使用它可以查询和管理分布式存储系统上的大数据集。

Hive提供了一种叫做HiveQL的类似SQL查询语言用来查询数据，HiveQL也允许熟悉MapReduce开发者开发自定义的mapper和reducer来处理内建的mapper和reducer无法完成的复杂的分析工作。

Hive的工作模式是提交一个任务，等到任务结束时被通知，而不是实时查询。

## Hive安装 ##

直接去[Hive官网](https://hive.apache.org/)下载最新的文件，解压。

运行bin目录里的hive文件，运行之前先启动hadoop，运行的时候可能会出现：

	Missing Hive CLI Jar ....
    
将hive解压出来的lib目录里的jline*.jar拷贝到$HADOOP/share/hadoop/yarn/lib里，同时将$HADOOP/share/hadoop/yarn/lib里的jline*.jar删除，重启hadoop。

再次运行，可能还会出现

	The reported blocks 2662 has reached the threshold 0.9990 of total blocks 2662. The number of live datanodes 1 has reached the minimum number 0. In safe mode extension. Safe mode will be turned off automatically in 0 seconds. ...
    
类似的问题，关闭hdfs的安全模式即可：

	hadoop dfsadmin -safemode leave
    
## 基本命令 ##

HiveQL就是模仿sql而创建的，以SQL的角度来介绍HiveQL。

### DDL操作 ###

创建表：

	hive> create table users(age INT, name STRING);
    
查看所有的表：

	hive> show tables;

查看以CLIENT开头的表：

	hive> show tables 'CLIENT.*';
    
表加列：

	hive> alter table users add columns(gender BOOLEAN);
    
改表名字：

	hive> alter table users rename to user;
    
删除表：

	hive> drop table user;
    
查看表的具体信息：

	hive> descibe user;
    hive> desc user;
    
### DML操作 ###

hive创建表的时候可以指定分隔符，由于hive操作的是hdfs，数据最终会存储在hdfs上，所以hdfs上的内容肯定是以某种分隔符分开各个列的。 hive默认的列分隔符是 **^A** 。 我们可以自定义自己的分隔符，在创建表的时候指定分隔符即可。

本地导入数据到hive：

	hive> load data local inpath 'localFile' overwrite into table users;
    
users表的结构只有2列，name和age，而且使用默认的分隔符。

比如本地文件的内容是这样的：

	format1^A11
	format2^A22
    
导入之后进行查询：

	hive> select * from users;
    
显示结果：

	OK
    format1	11
    format2	22
    Time taken: 0.362 seconds, Fetched: 2 row(s)


在创建表的时候可以指定列分隔符和数组分隔符：

	hive> create table users(name string, age int)
     > ROW FORMAT DELIMITED
     > FIELDS TERMINATED BY '\t'
     > COLLECTION ITEMS TERMINATED BY ',';


导入数据还有几个参数：

local参数意味着从本地加载文件，如果没有local参数，那表示从hdfs加载文件。

关键字overwrite意味着当前表中已经存在的数据将会被删除掉，没有overwrite关键字，表示数据是追加，追加到原先数据集里面。


插入数据，插入数据后会起一个map reduce job去跑插入的数据：

	hive> insert into table users values('formatgogo', 222);

带条件的查询数据：
	
    hive> select * from users where age = 11;
    
group by查询：

	hive> select age, count(1) from users group by age;

partition的使用，以部门表为例，用type进行partition：

	hive> create table dept(name STRING) partitioned by (type INT);
   
以2个文件为例，dept1.txt：

	dept1^A1
    dept11^A1
    dept111^A1
    dept1111^A1
    dept11111^A1
    dept111111^A1
    
dept2.txt

	dept2^A2
    dept22^A2
    dept222^A2
    dept2222^A2
    dept22222^A2
    dept222222^A2

使用partition之后，导入数据的时候需要指定对应的partition：

	hive> load data local inpath '$PATH/dept1.txt' overwrite into table dept partition(type=1);
    hive> load data local inpath '$PATH/dept2.txt' overwrite into table dept partition(type=2);
    
    hive> select * from dept;
  
结果：
  
    OK
    dept1	1
    dept11	1
    dept111	1
    dept1111	1
    dept11111	1
    dept111111	1
    dept2	2
    dept22	2
    dept222	2
    dept2222	2
    dept22222	2
    dept222222	2
    Time taken: 0.114 seconds, Fetched: 12 row(s)


insert可以将数据导出到指定目录，将users表导入到本地文件。 去掉local关键字表示导出到hdfs目录：

	hive> insert overwrite local directory 'localFileName' select * from users;
	
### hive存储在hdfs的位置 ###

进入hive控制台之后，可以使用：

	hive> set hive.metastore.warehouse.dir;
    
查看hive存储在hdfs的位置，默认是存在 /user/hive/warehouse 目录。

之前的users表，会存储在/user/hive/warehouse/user目录里。

有partition的表会存在的不同位置，比如之前的dept表的type为1和2的分别存储在 /user/hive/warehouse/dept/type=1 和 /user/hive/warehouse/dept/type=2。

## 数据类型 ##

hive中的数据类型分2种，简单类型和复杂类型。

简单类型有以下几种：TINYINT, SMALLINT, INT, BIGINT, BOOLEAN, FLOAT, DOUBLE, STRING。

复杂类型有以下几种：Structs(结构体，学过C都知道)，MAPS(key-value键值对)，Arrays(数组类型，数组内的元素类型都必须一致)


简单类型就不分析了，来看一下复杂类型的使用：

### Structs ###

	hive> create table employee(id INT, info struct<name:STRING, age:INT>)
    	  > ROW FORMAT DELIMITED FIELDS TERMINATED BY ',' 
          > COLLECTION ITEMS TERMINATED BY ':'; 
        
要导入的数据：

	1,format:11
	2,fj:22
	3,formatfj:33
    
导入数据：

	hive> load data local inpath 'localFile' overwrite into table employee;
    
查询：

	select * from employee;
    
    OK
    1	{"name":"format","age":11}
    2	{"name":"fj","age":22}
    3	{"name":"formatfj","age":33}
    Time taken: 0.042 seconds, Fetched: 3 row(s)

	hive> select info.name from employee;
    
    OK
    format
    fj
    formatfj
    Time taken: 0.061 seconds, Fetched: 3 row(s)

### Maps ###

	hive> create table lessons(id string, score map<string, int>)
    	> ROW FORMAT DELIMITED
        > FIELDS TERMINATED BY '\t'
        > COLLECTION ITEMS TERMINATED BY ','
        > MAP KEYS TERMINATED BY ':';
        
要导入的数据：

	1       chinese:80,english:60,math:70  
    2       computer:60,chemistry:80   

导入数据：

	hive> load data local inpath 'localFile' overwrite into table lessons;

查询：

	hive> select score['chinese'] from lessions;
    
    OK
    80
    NULL
    Time taken: 0.05 seconds, Fetched: 2 row(s)
    
    hive> select * from lessons;
    OK
    1	{"chinese":80,"english":60,"math":70}
    2	{"computer":60,"chemistry":80}
    Time taken: 0.032 seconds, Fetched: 2 row(s)

### Arrays ###

	hive> create table student(name string, hobby_list array<STRING>)
    	> ROW FORMAT DELIMITED
        > FIELDS TERMINATED BY ','
        > COLLECTION ITEMS TERMINATED BY ':';
        
要导入的数据：

	format,basketball:football:swimming
    fj,coding:running
    
导入数据：

	hive> load data local inpath 'localFile' overwrite into table student;

查询(数组下标没对应的值的话返回NULL)：

	hive> select * from student;
    
    OK
    format	["basketball","football","swimming"]
    fj	["coding","running"]
    Time taken: 0.041 seconds, Fetched: 2 row(s)

	hive> select hobby_list[2] from student;

	OK
    swimming
    NULL
    Time taken: 0.07 seconds, Fetched: 2 row(s)


