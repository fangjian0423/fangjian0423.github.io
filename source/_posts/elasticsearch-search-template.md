title: elasticsearch查询模板
date: 2015-11-07 16:04:25
tags:
- elasticsearch
- big data
categories:
- elasticsearch
description: 近在公司又用到了elasticsearch，也用到了查询模板，顺便写篇文章记录一下查询模板的使用 ...

---------------


最近在公司又用到了elasticsearch，也用到了查询模板，顺便写篇文章记录一下查询模板的使用。

以1个需求为例讲解es模板的使用：


**页面上某个按钮在一段时间内的点击次数统计，并且可以以小时，天，月为单位进行汇总，并且需要去重。**


创建索引，只定义3个字段，user_id, user_name和create_time:

	-POST /$ES/event_index

    {
      "mappings": {
        "event": {
          "_ttl": {
            "enabled": false
          },
          "_timestamp": {
            "enabled": true,
            "format": "yyyy-MM-dd HH:mm:ss"
          },
          "properties": {
            "user_id": {
              "type": "string",
              "store": "no",
              "index": "not_analyzed"
            },
            "create_time": {
              "type": "date",
              "store": "no",
              "index": "not_analyzed",
              "format": "yyyy-MM-dd HH:mm:ss"
            },
            "user_name": {
              "type": "string",
              "store": "no"
            }
          }
        }
      }
    }

定义对应的查询模板，模板名字stats，使用了Cardinality和DateHistogram这两个Aggregation
，其中Date Histogram嵌套在Cardinality里。在定义模板的时候，{ { } } 的表示是个参数，需要调用模板的时候传递进来:

  	-POST /$ES/_search/template/stats
    {
        "template": {
            "query": {
                "bool": {
                    "must": [
                        {
                            "range": {
                                "create_time": {
                                    "gte": "{{earliest}}",
                                    "lte": "{{latest}}"
                                }
                            }
                        }
                    ]
                }
            },
            "size": 0,
            "aggs": {
                "stats_data": {
                    "date_histogram": {
                        "field": "create_time",
                        "interval": "{{interval}}"
                    },
                    "aggs": {
                        "time": {
                            "cardinality": {
                                "field": "user_id"
                            }
                        }
                    }
                }
            }
        }
	}

Cardinality Aggregation的作用就是类似sql中的distinct，去重。

Date Histogram Aggregation的作用是根据时间进行统计。内部有个interval属性表面统计的范畴。


下面加几条数据到event_index里：


	-POST $ES/event_index/event
    {
        "user_id": "1",
        "user_name": "format1",
        "create_time": "2015-11-07 12:00:00"
    }

	-POST $ES/event_index/event
	{
        "user_id": "2",
        "user_name": "format2",
        "create_time": "2015-11-07 13:30:00"
    }
    
    -POST $ES/event_index/event
    {
        "user_id": "3",
        "user_name": "format3",
        "create_time": "2015-11-07 13:30:00"
    }
    
    -POST $ES/event_index/event
    {
        "user_id": "1",
        "user_name": "format1",
        "create_time": "2015-11-07 13:50:00"
    }
    
    -POST $ES/event_index/event
    {
        "user_id": "1",
        "user_name": "format1",
        "create_time": "2015-11-07 13:55:00"
	}

11-07 12-13点有1条数据，1个用户
11-07 13-14点有4条数据，3个用户

    
使用模板查询：

	curl -XGET "$ES/event_index/_search/template" -d'{
      "template": { "id": "stats" }, 
      "params": { "earliest": "2015-11-07 00:00:00", "latest": "2015-11-07 23:59:59", "interval": "hour" }
    }'	

结果：

    {
        "took": 3,
        "timed_out": false,
        "_shards": {
            "total": 5,
            "successful": 5,
            "failed": 0
        },
        "hits": {
            "total": 5,
            "max_score": 0,
            "hits": []
        },
        "aggregations": {
            "stats_data": {
                "buckets": [
                    {
                        "key_as_string": "2015-11-07 12:00:00",
                        "key": 1446897600000,
                        "doc_count": 1,
                        "time": {
                            "value": 1
                        }
                    },
                    {
                        "key_as_string": "2015-11-07 13:00:00",
                        "key": 1446901200000,
                        "doc_count": 4,
                        "time": {
                            "value": 3
                        }
                    }
                ]
            }
        }
    }

12点-13点的只有1条数据，1个用户。13-14点的有4条数据，3个用户。


以天(day)统计：

	curl -XGET "$ES/event_index/_search/template" -d'{
      "template": { "id": "stats" }, 
      "params": { "earliest": "2015-11-07 00:00:00", "latest": "2015-11-07 23:59:59", "interval": "day" }
    }'	

结果：

	{
        "took": 4,
        "timed_out": false,
        "_shards": {
            "total": 5,
            "successful": 5,
            "failed": 0
        },
        "hits": {
            "total": 5,
            "max_score": 0,
            "hits": []
        },
        "aggregations": {
            "stats_data": {
                "buckets": [
                    {
                        "key_as_string": "2015-11-07 00:00:00",
                        "key": 1446854400000,
                        "doc_count": 5,
                        "time": {
                            "value": 3
                        }
                    }
                ]
            }
        }
    }

11-07这一天有5条数据，3个用户。


本文只是简单说明了es查询模板的使用，也简单使用了2个aggregation。更多内容可以去官网查看相关资料。
