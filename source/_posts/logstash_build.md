title: logstash搭建日志追踪系统
date: 2014-11-01 22:40:05
tags:
- logstash
- elasticsearch
- kibana
categories:
- elasticsearch
description: logstash是一个用来管理事件和日志的工具，它的作用是收集日志，解析日志，存储日志为以后使用 ...

----------------

## 前言 ##
开始博客之前，首先看个问题：
作为一只程序猿，写的代码的过程需要加入一些日志信息，这些日志信息包括debug调试信息，异常记录日志等。  Java猿一般都是使用log4j，logback等第三方库记录日志。 那么问题来了，挖掘机到底哪家强？......  扯个淡，那么问题来了，如果我们想看日志信息，怎么办， ssh到服务器上，vim然后查询。每次都这样，是不是很蛋疼 = =； 还有另外一个问题，如果我们想分析、追踪日志，或者找关键字(分词后的关键字)，这样简简单单看日志文件是不可能的。 因此，我们就需要开源力量了！

## logstash介绍##
摘自官网上的一句话：[logstash](http://logstash.net/) is a tool for managing events and logs. You can use it to collect logs, parse them, and store them for later use (like, for searching)。logstash是一个用来管理事件和日志的工具，它的作用是收集日志，解析日志，存储日志为以后使用。

官网上有[tutorials](http://logstash.net/docs/1.4.2/tutorials/getting-started-with-logstash)。 本文也就是对tutorials做一个总结。

## logstash日志追踪系统搭建过程 ##
要搭建logstash日志追踪系统需要以下几个环境：
1. JDK
2. logstash
3. [elasticsearch](http://www.elasticsearch.org)

没有JDK的小伙伴首先先去[下载](http://www.oracle.com/technetwork/java/javase/downloads/index.html?ssSourceSiteId=ocomen)吧。 logstash和elasticsearch都先下载过来吧~。

### logstash环境搭建 ###

首先先进入logstash的bin目录建立一个logstash.conf配置文件：

	input { stdin { } }
    output {
      stdout { codec => rubydebug }
    }

然后执行：
	 
     ./logstash -f logstash.conf     
这时控制台等待输入内容，我们输入hello world，这个时候控制台会打印出：

	{
       "message" => "hello world",
       "@version" => "1",
       "@timestamp" => "2014-11-01T12:38:17.217Z",
          "host" => "format-2.local"
	}

这个说明我们的logstash本地配置成功了。

**配置文件有2个内容组成，input和output，其实还有2个配置：filter和codec。**
这就是logstash内部的一个叫做**事件处理管道(pipeline)**的核心概念的3大组成部分：
input：
生成事件(logstash中的事件是由队列实现的，这个队列由ruby的SizedQueue实现)的数据来源，常见的有file(文件)、syslog(系统日志)、redis(缓存系统)、[lumberjack(lumberjack协议)](https://github.com/elasticsearch/logstash-forwarder)。

filter：
修改事件内容，常见的filter有grok(常用，解析文本并结构化地存储下来，用来处理没有结构的文本)、mutate、drop、clone、geoip

output：
展现结果，常见的有elasticsearch(搜索引擎)、file、graphite、statsd

codec：
可以作为input或output的一部分，主要用来处理日志过程中产生的消息，常见的codec有json、rubydebug

现在我们回过头看来我们的logstash.conf配置文件，只配置了input和output，其中input由一个stdin组成，这个stdin没有任何参数，output由stdout组成，这个stdout由codec参数，且使用了rebydebug，因此控制台打印出的信息是reby的对象格式。  我们把codec改成json的话，将会打印出以下内容：

	{"message":"hello world","@version":"1","@timestamp":"2014-11-01T13:16:38.221Z","host":"format-2.local"}

### elasticsearch环境搭建 ###

elasticsearch的环境搭建比较简单，download elasticsearch之后进入bin目录，执行：
	
    ./elasticsearch
之后打开浏览器进入http://localhost:9200/，发现有一串json文本就表示elasticsearch服务器已启。 但是貌似没有发现什么界面，是不是很不友好= =。

elasticsearch支持插件功能，我们使用[kibana插件](http://www.elasticsearch.org/overview/kibana/)，下载之后修改config.js文件，把elasticsearch对应的地址改成elasticsearch服务器地址，然后把kibana解压出来的所有文件放到$elasticsearch_home/plugins_/kibana/_site/目录中。
之后打开[http://localhost:9200/_plugin/kibana](http://localhost:9200/_plugin/kibana)

![](http://format-blog-image.qiniudn.com/logstash1.png)
![](http://format-blog-image.qiniudn.com/logstash2.png)

### logstash整合elasticsearch ###

配置完logstash和elasticsearch之后，整合一下这两个框架。
logstash配置文件(input、output可以配置多个)：

	input { stdin { } }
    output {
      stdout { codec => json }
      elasticsearch { host => localhost }
    }

然后重新启动logstash，控制台输入hello elasticsearch，刷新kibana页面：
![](http://format-blog-image.qiniudn.com/logstash3.png)

logstash日志追踪系统搭建完毕。

## logstash的实际应用 ##
以log4j为例。

logstash配置：

    input {
      stdin { }
      log4j {
         mode => "server"
         host  => "127.0.0.1"
         port => 56789
         type => "log4j"
      }
    }
    output {
      stdout { codec => rubydebug }
      elasticsearch { host => localhost }
    }

测试类：

	public class LogTest {

        private Logger logger = Logger.getLogger(DebugLogger.class);

        @Before
        public void setUp() {
            PropertyConfigurator.configure(LogTest.class.getClassLoader().getResourceAsStream("log4j.properties"));
        }

        @Test
        public void testLog() {
            logger.debug("hello logstash, this is a message from log4j");
        }

        @Test
        public void testException() {
            logger.error("error", new TestException("sorry, error"));
        }

}

log4j配置：
	
    log4j.appender.stdout=org.apache.log4j.ConsoleAppender
    log4j.appender.stdout.layout=org.apache.log4j.PatternLayout
    log4j.appender.stdout.layout.ConversionPattern=%d %p %t %c : %m%n

    log4j.appender.file=org.apache.log4j.RollingFileAppender
    log4j.appender.file.file=/Users/fangjian/Develop/log_file/test_log.log
    log4j.appender.file.maxFileSize=1024
    log4j.appender.file.layout=org.apache.log4j.PatternLayout
    log4j.appender.file.layout.ConversionPattern=%d %p %t %c : %m%n

    # logstash配置
    log4j.appender.logstash=org.apache.log4j.net.SocketAppender
    log4j.appender.logstash.port=56789
    log4j.appender.logstash.remoteHost=127.0.0.1

    log4j.rootLogger=debug,stdout,file,logstash


2个test方法跑完之后，刷新kibana界面：
![](http://format-blog-image.qiniudn.com/logstash4.png)

## 总结 ##
本文仅仅只是对logstash的搭建做一个总结，包括logstash内部的结构，还有一些配置语言的介绍都没有非常详细的解释，如果读者有兴趣，可以自行查阅相关资料。

参考资料：
[http://logstash.net/](http://logstash.net/)
[http://blog.yeradis.com/2013/10/logstash-and-apache-log4j-or-how-to.html](http://blog.yeradis.com/2013/10/logstash-and-apache-log4j-or-how-to.html)
[http://www.cnblogs.com/buzzlight/p/logstash_elasticsearch_kibana_log.html](http://www.cnblogs.com/buzzlight/p/logstash_elasticsearch_kibana_log.html)


