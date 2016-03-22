title: Elasticsearch入门
date: 2015-07-12 03:06:59
tags: 
- elasticsearch
- big data
categories:
- elasticsearch
description: Elasticsearch是一个基于Lucene的搜索服务器。它提供了一个分布式多用户能力的全文搜索引擎，使用RESTful web暴露接口 ...

---------------

之前搭建logstash的时候使用过elasticsearch。 刚好最近在公司也用到了es，写篇水文记录一下也当做笔记吧。

[Elasticsearch](https://www.elastic.co/products/elasticsearch)是一个基于Lucene的搜索服务器。它提供了一个分布式多用户能力的全文搜索引擎，使用RESTful web暴露接口。

它有许多特性，比如以下几个属性：

1.实时数据 
2.实时分析
3.分布式设计
4.高可用性
5.全文搜索
6.面向文档

## 索引 ##

索引Index是es中的一个存储数据的地方。相当于关系型数据库中的数据库。

创建一个员工索引的例子如下，创建索引还有很多选项，就不一一说明了：

	POST $HOST/employee
    
    {
      "mappings": {
        "employee": {
          "_ttl": {
            "enabled": true,
            "default": "5d"
          },
          "_timestamp": {
            "enabled": true,
            "format": "yyyy-MM-dd HH:mm:ss"
          },
          "properties": {
            "name": {
              "type": "string",
              "store": "no",
              "index": "not_analyzed",
              "index_options": "docs"
            },
            "birth_date": {
              "type": "date",
              "store": "no",
              "index": "not_analyzed",
              "index_options": "docs",
              "format": "yyyy-MM-dd HH:mm:ss"
            },
            "age": {
              "type": "date",
              "store": "no",
              "index": "not_analyzed",
              "index_options": "docs",
              "format": "yyyy-MM-dd HH:mm:ss"
            }
          }
        }
      }
    }
    
索引创建完之后还可以修改(添加一个hobby属性)，需要注意的是，修改mapping不允许修改属性的类型：

	PUT $HOST/employee/employee/_mapping
    
    {
    "employee": {
        "properties": {
            "name": {
                "type": "string",
                "store": "no",
                "index": "not_analyzed",
                "index_options": "docs"
            },
            "birth_date": {
                "type": "date",
                "store": "no",
                "index": "not_analyzed",
                "index_options": "docs",
                "format": "yyyy-MM-dd HH:mm:ss"
            },
            "age": {
                "type": "date",
                "store": "no",
                "index": "not_analyzed",
                "index_options": "docs",
                "format": "yyyy-MM-dd HH:mm:ss"
            },
            "hobby" : {
            	"type" : "string",
                "index_options": "docs"
            }
		}
	}
	}

## 文档 ##

es存储的数据叫做文档，文档存储在索引中。 每个文档都有4个元数据，分别是_id, _type，_index和_version。

_id代表文档的唯一标识符。

_type表示文档代表的对象种类。

_index表示文档存储在哪个索引。

_version表示文档的版本，文档被修改过一次，_version就会+1。

在员工索引中创建文档：

	POST $HOST/employee/employee
    
    {
        "name": "format",
        "age": 100,
        "birth_date": "1900-01-01 00:00:00"
    }
    
返回：

    {
        "_index": "employee",
        "_type": "employee",
        "_id": "AU5-epuwslU6QVfs_UoX",
        "_version": 1,
        "created": true
	}
    
修改文档：

	POST $HOST/employee/employee/AU5-epuwslU6QVfs_UoX
    
    {
        "name": "format",
        "age": 200,
        "birth_date": "1900-01-01 00:00:00"
    }
    
返回：

	{
        "_index": "employee",
        "_type": "employee",
        "_id": "AU5-epuwslU6QVfs_UoX",
        "_version": 2,
        "created": false
	}
    
删除文档：

	DELETE $HOST/employee/employee/AU5-epuwslU6QVfs_UoX
    
返回：

    {
        "found": true,
        "_index": "employee",
        "_type": "employee",
        "_id": "AU5-epuwslU6QVfs_UoX",
        "_version": 3
    }
    
## 总结 ##

写了篇水文记录一下es，es还有很多很强大的功能，比如一些query，filter，aggregations等。官方文档上已经写的非常清楚了。这里就不讲了。  - -||
