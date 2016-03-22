title: Kafka介绍
date: 2016-01-13 23:54:51
tags:
- big data
- log
categories:
- kafka
description: Kafka是一个分布式的发布-订阅消息系统(Producer-Consumer)，是一种快速、可扩展的、分区的和可复制的日志服务。Kafka中有几个概念，分别是Topic，Broker，Producer，Consumer等 ...

--------------

## Kafka介绍 ##

Kafka是一个分布式的发布-订阅消息系统(Producer-Consumer)，是一种快速、可扩展的、分区的和可复制的日志服务。

kafka中的几个概念：

Topic：用来区别各种message。比如A系统的所有message的Topic为A，B系统的所有message的Topic为B。

Broker：已发布的消息保存在一组服务器中，这组服务器就被称为Broker或Kafka集群。

Producer：生产者，用于发布消息，发布消息到kafka broker。

Consumer：消费者，订阅消息，订阅kafka broker中的已经被发布的消息。

下图是几个概念的说明：

producer发布消息到kafka cluster(也就是kafka broker)，然后发布后的这些消息被consumer订阅。

从图中也可以看出来，kafka支持多个producer和多个consumer。

![](http://kafka.apache.org/images/producer_consumer.png)

## Kafka存储机制 ##

Partition：Kafka中每个Topic都会有一个或多个Partition，由于kafka将数据直接写到硬盘里，这里的Partition对应一个文件夹，文件夹下存储这个Partition的所有消息和索引。如果有2个Topic，分别有3个和4个Partition。那么总共有7个文件夹。Kafka内部会根据一个算法，根据消息得出一个值，然后根据这个值放入对应的partition目录中的段文件里。

比如在一台机器上创建partition为3，topic为test01和partition为4，topic为test02的2个topic。

创建完之后 /tmp/kafka-logs中就会有7个文件夹，分别是 

test01-0
test01-1
test01-2
test02-0
test02-1
test02-2
test02-3

Segment：组成Partiton的组件。一个Partition代表一个文件夹，而Segment则是这个文件夹下的各个文件。每个Segmenet文件有大小限制，在配置文件中用log.segment.bytes配置。

	log.segment.bytes=1073741824
    
当文件的大小超过1073741824字节的时候，会创建第一个段文件。需要注意的是这里每个段文件中的消息数量不一定相等，因为虽然他们的字节数一样，但是每个消息的字节数是不一样的，所以每个段文件中的消息数量不一定相等。

每个段文件由2部分组成，分别是index file和log file，表示索引文件和日志(数据)文件。这2个文件一一对应。

第一个segment文件从0开始，后续每个segment文件名是上一个segment文件的最后一条message的offset值，数值最大为64位long大小，19位数字字符长度，没有数字用0填充。

下面是做的一个例子，partition和replication-factor都为1，每个segmenet文件的大小是5M。有500000条message，一共生成了4对文件，这里00000000000000137200.log文件表示是00000000000000000000.log中存储了137199个message，这个文件开始存储第137200个message。

00000000000000000000.index
00000000000000000000.log

00000000000000137200.index
00000000000000137200.log

00000000000000271600.index
00000000000000271600.log

00000000000000406000.index
00000000000000406000.log

offset：用来标识message在partition中的下标，用来定位message。

Kafka内部存储结构可以参考[Kafka文件存储机制那些事](http://tech.meituan.com/kafka-fs-design-theory.html)文章里的讲解。


一个kafka producer例子：

	import java.util.Properties
	import kafka.producer.{KeyedMessage, Producer, ProducerConfig}

	object TestProducer extends App {

      val events = 500000
      val props = new Properties()
      val brokers = "localhost:9092"
      props.put("metadata.broker.list", brokers)
      props.put("serializer.class", "kafka.serializer.StringEncoder")
      props.put("producer.type", "async")

      val config = new ProducerConfig(props)

      val topic = "format03"

      val producer = new Producer[String, String](config)

      for(nEvents <- Range(0, events)) {
        val msg = "Message" + nEvents
        val data = new KeyedMessage[String, String](topic, msg)
        producer.send(data)
      }

      producer.close()

    }


## 参考资料 ##

[Kafka文件存储机制那些事](http://tech.meituan.com/kafka-fs-design-theory.html)
[Kafka剖析（一）：Kafka背景及架构介绍](http://www.infoq.com/cn/articles/kafka-analysis-part-1/)
[Kafka设计解析（二）：Kafka High Availability （上）](http://www.infoq.com/cn/articles/kafka-analysis-part-2/)
[Kafka设计解析（三）：Kafka High Availability （下）](http://www.infoq.com/cn/articles/kafka-analysis-part-3/)




