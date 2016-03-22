title: Spark Streaming编程指南笔记
date: 2016-02-10 01:36:17
tags:
- spark
- big data
categories:
- spark
description: Spark Streaming是Spark核心API的扩展，用于处理实时数据流。Spark Streaming处理的数据源可以是Kafka，Flume，Twitter，ZeroMQ，Kinesis或者Tcp Sockets ...

--------------

## 概述 ##

Spark Streaming是Spark核心API的扩展，用于处理实时数据流。Spark Streaming处理的数据源可以是Kafka，Flume，Twitter，ZeroMQ，Kinesis或者Tcp Sockets，这些数据可以使用map，reduce，join，window方法进行处转换，还可以直接使用Spark内置的机器学习算法，图算法包来处理数据。

![](http://spark.apache.org/docs/latest/img/streaming-arch.png)

最终处理后的数据可以存入文件系统，数据库。

Spark Streaming内部接收到实时数据之后，会把数据分成几个批次，这些批次数据会被Spark引擎处理并生成各个批次的结果。

![](http://spark.apache.org/docs/latest/img/streaming-flow.png)


Spark Streaming提供了一个叫做**discretized stream 或者 DStream**的抽象概念，表示一段连续的数据流。DStream会在数据源中的数据流中创建，或者在别的DStream中使用类似map，join方法创建。一个DStream表示一个RDD序列。

## 一个快速例子 ##

以一个TCP Socket监听接收数据，并计算单词的个数为例子讲解。

首先，需要import Spark Streaming中的一些类和StreamingContext中的一些隐式转换。我们会创建一个带有2个线程，1秒一个批次的StreamingContext。

    import org.apache.spark.SparkConf
    import org.apache.spark.streaming.{Seconds, StreamingContext}

    object SparkStreamTest extends App {

      val conf = new SparkConf().setMaster("local[2]").setAppName("NetworkWordCount")

	  // 创建一个StreamingContext，每1秒钟处理一次计算程序
      val ssc = new StreamingContext(conf, Seconds(1))

      // 使用StreamingContext创建DStream，DStream表示TCP源中的流数据. lines这个DStream表示接收到的服务器数据，每一行都是文本
      val lines = ssc.socketTextStream("localhost", 9999)

      // 使用flatMap将每一行中的文本转换成每个单词，并产生一个新的DStream。
      val words = lines.flatMap(_.split(" "))

      // 使用map方法将每个单词转换成tuple
      val pairs = words.map(word => (word, 1))

      // 使用reduceByKey计算出每个单词的出现次数
      val wordCounts = pairs.reduceByKey(_ + _)

      wordCounts.print()

      ssc.start() // 开始计算
      ssc.awaitTermination() // 等待计算结束
    }
    
在运行这段代码之前，首先先起一个netcat服务：

	nc -lk 9999
    
之后比如输入hello world之后，控制台会打印出如下数据：

	-------------------------------------------
    Time: 1454684570000 ms
    -------------------------------------------
    (hello,1)
    (world,1)

## 基础概念  ##

### Linking(SparkStreaming的连接) ###

写Spark Streaming程序需要一些依赖。使用maven的话加入以下依赖：

    <dependency>
        <groupId>org.apache.spark</groupId>
        <artifactId>spark-streaming_2.10</artifactId>
        <version>1.6.0</version>
    </dependency>

使用sbt的话，加入以下依赖：

	libraryDependencies += "org.apache.spark" % "spark-streaming_2.10" % "1.6.0"
    
SparkStreaming核心不提供一些数据源的依赖，需要手动添加，一些数据源对应的Artifact如下：
    
| 数据源 | Artifact|
|:----:|:----:|
|  Kafka   |  spark-streaming-kafka_2.10  |
|  Flume   |  spark-streaming-flume_2.10   |
|  Kinesis   |  spark-streaming-kinesis-asl_2.10 [Amazon Software License]   |
|  Twitter   |  spark-streaming-twitter_2.10   |
|  ZeroMQ   |  spark-streaming-zeromq_2.10   |
|  MQTT   |  spark-streaming-mqtt_2.10		|

### StreamingContext的初始化  ###

StreamingContext的创建是Spark Streaming程序中最重要的一环。


可以根据SparkConf对象创建出StreamingContext对象：

    import org.apache.spark._
    import org.apache.spark.streaming._

    val conf = new SparkConf().setAppName(appName).setMaster(master)
    val ssc = new StreamingContext(conf, Seconds(1))

appName参数是应用程序的名字，在cluster UI中显示的就是这个名字。master参数的意义跟spark中master参数的意义是一样的。

StreamingContext内部会创建SparkContext，可以使用StreamingContext内部的sparkContext获得。

	ssc.sparkContext // 得到SparkContext
    
StreamingContext也可以根据SparkContext创建：

    import org.apache.spark.streaming._

    val sc = ...                // existing SparkContext
    val ssc = new StreamingContext(sc, Seconds(1))
    
StreamingContext创建之后，可以做以下几点：

1.创建DStreams定义数据源
2.使用DStreams的transformation和output operations用于计算
3.使用streamingContext的start方法接收数据
4.使用streamingContext的awaitTermination方法等待处理结果
5.可以使用streamingContext的stop方法停止程序

一些需要注意的点：

1.context开始启动之后，一些streaming的计算不允许发生
2.context停掉之后不能重启
3.一个JVM在同一时刻只能有一个StreamingContext可以激活
4.StreamingContext中的stop方法内部也会stop SparkContext。如果只想stop StreamingContext，那么调用stop方法的时候参数设置为false
5.一个SparkContext可以用来创建多个StreamingContexts，只要上一个StreamingContext在下一个StreamingContext创建之前停掉

### Discretized Streams (DStreams) ###

DStreams和Discretized Streams在Spark Streaming中代表相同的意思：

1.一段连续的数据流
2.数据源中接收到的数据流
3.使用transforming处理过的流数据

Spark内部一个DStream表示一段连续的RDD。DStream中每段RDD表示一段时间内的RDD，效果如下：

![](http://spark.apache.org/docs/latest/img/streaming-dstream.png)

DStream可以使用一些transformation操作将内部的RDD转换成另外一种RDD。比如之前的一个单词统计例子中就将一行文本的DStream转换成每个单词的DStream，过程如下：

![](http://spark.apache.org/docs/latest/img/streaming-dstream-ops.png)

### Input DStreams and Receivers(数据源和接收器) ###

Input DStreams是DStreams从streaming source中接收到的输入流数据。在之前分析的一个单词统计例子中，lines就是个Input DStream，表示接收到的服务器数据，每一行都是文本。

每一个Input DStream(除了file stream)都会关联一个Receiver对象，这个Receiver对象的作用是接收数据源中的数据并存储在内存中。

Spark Streaming提供了两种类型的内置数据源：

1.基础数据源。可以直接使用StreamingContext的API，比如文件系统，socket连接，Akka。
2.高级数据源。比如Flume，Kafka，Kinesis，Twitter等可以使用工具类的数据源。使用这些数据源需要对应的依赖，在Linking章节中已经介绍过。

如果想在streaming程序中并行地接收多个数据源，需要创建多个Input DStream，有个多个Input DStream的话那就会对应地有多个Receiver。但是需要记住的是，Spark的worker/executor模式是一个相当耗时的任务，因此服务器的配置需要够好才能支撑多个Input DStream。

一些注意点：

1.当本地跑Spark Streaming程序的时候，不要使用"local"或者"local[1]"设置master URL。因为这两种master URL只会使用1个线程。当使用比如Flume，Kafka，socket这些数据源的时候，因为只有一个线程跑receiver接收数据，那么没有其他线程去处理接收后的数据了。所以，当在本地跑Spark Streaming程序的时候，需要将master URL设置为local[n]，n需要大于receiver的个数。
2.服务器的核数需要大于receiver的个数。否则程序只会接收数据，而不会处理数据。

#### Basic Sources(基础数据源) ####

基础数据源刚刚分析过，StreamingContext的API可以使用如文件系统，socket连接，Akka作为输入源。socket连接本文以开始的例子中已经使用过了。

文件系统的输入源会读取文件或任何支持HDFS API(比如HDFS，S3，NFS)的文件系统的数据：

	streamingContext.fileStream[KeyClass, ValueClass, InputFormatClass](dataDirectory)
    
Spark Streaming会监测dataDirectory目录并且会处理这个目录中新创建的文件(老文件写新数据的话不会被支持)。使用文件数据源还需要这几点：

1.所有文件的数据格式必须相同
2.dataDirectory目录中的文件必须是新创建的，也可以是从别的目录move进来的
3.文件内部的数据更改之后，新更改的数据不会被处理

对于简单的文件，可以使用streamingContext的textFileStream方法处理。

#### Advanced Sources(高级数据源) ####

高级数据源需要一些非Spark依赖。Spark Streaming把创建DStream的API移到了各自的API里。如果想创建一个使用Twitter的数据源，需要做以下三步：

1.添加对应的Twitter依赖spark-streaming-twitter_2.10到项目里
2.import这个类TwitterUtils，使用TwitterUtils.createStream创建DStream
3.部署

#### Custom Sources(自定义数据源) ####

要实现一个自定义的数据源，需要实现一个自定义的receiver


#### Receiver Reliability(接收器的可靠性) ####

基于可靠性的数据源分为两种。

1.可靠的接收器(Reliable Receiver)：一个可靠的接收器接收到数据之后会给数据源发送消息表示自己已经接收到数据
2.不可靠的接收器(Unreliable Receiver)：一个不可靠的接收器不会发送消息给数据源。

想写出一个可靠的接收器可以参考 [http://spark.apache.org/docs/latest/streaming-custom-receivers.html](http://spark.apache.org/docs/latest/streaming-custom-receivers.html)

### DStreams的Transformations操作 ###

DStream的Transformations操作跟RDD的Transformations操作类似，

| Transformation | 含义|
|:----:|:----:|
|  map(func)   |  根据func函数生成一个新的DStream  |
|  flatMap(func)   |  跟map方法类似，但是每一项可以返回多个值。func函数的返回值是一个集合   |
|  filter(func)   |  根据func函数返回true的数据集   |
|  repartition(numPartitions)   |   重新给 DStream 分区   |
|  union(otherStream)   |  取2个DStream的并集，得到一个新的DStream  |
|  count()   |  返回一个新的只有一个元素的DStream，这个元素就是DStream中的所有RDD的个数  |
|  reduce(func)   |  返回一个新的只有一个元素的DStream，这个元素就是DStream中的所有RDD通过func函数聚合得到的结果  |
|  countByValue()   |  如果DStream的类型为K，那么返回一个新的DStream，这个新的DStream中的元素类型是(K, Long)，K是原先DStream的值，Long表示这个Key有多少次  |
|  reduceByKey(func, [numTasks])   |  本文的例子使用过这个方法，对于是键值对(K,V)的DStream，返回一个新的DStream以K为键，各个value使用func函数操作得到的聚合结果为value  |
|  join(otherStream, [numTasks])   |  基于(K, V)键值对的DStream，如果对(K, W)的键值对DStream使用join操作，可以产生(K, (V, W))键值对的DStream   |
|  cogroup(otherStream, [numTasks])   |  跟join方法类似，不过是基于(K, V)的DStream，cogroup基于(K, W)的DStream，产生(K, (Seq[V], Seq[W]))的DStream  |
|  transform(func)   |  基于DStream中的每个RDD调用func函数，func函数的参数是个RDD，返回值也是个RDD   |
|  updateStateByKey(func)   |  对于每个key都会调用func函数处理先前的状态和所有新的状态。比如就可以用来做累加，这个方法跟reduceByKey类似，但比它更加灵活   |


#### UpdateStateByKey操作 ####

使用UpdateStateByKey方法需要做以下两步：

1.定义状态：状态可以是任意的数据类型
2.定义状态更新函数：这个函数需要根据输入流把先前的状态和所有新的状态

不管有没有新数据进来，在每个批次中，Spark都会对所有存在的key调用func方法，如果func函数返回None，那么key-value键值对不会被处理。

以一个例子来讲解updateStateByKey方法，这个例子会统计每个单词的个数在一个文本输入流里：

runningCount是一个状态并且是Int类型，所以这个状态的类型是Int，runningCount是先前的状态，newValues是所有新的状态，是一个集合，函数如下：

    def updateFunction(newValues: Seq[Int], runningCount: Option[Int]): Option[Int] = {
        val newCount = ...  // add the new values with the previous running count to get the new count
        Some(newCount)
    }

updateStateByKey方法的调用：

	val runningCounts = pairs.updateStateByKey[Int](updateFunction _)

#### Transform操作 ####

Transform操作针对的是RDD-RDD的操作，所以可以用来处理那些没有在DStream API中暴露的处理任意的RDD操作。比如在DStream中的每次批次没有join rdd的API，所以可以使用transform操作：

    val spamInfoRDD = ssc.sparkContext.newAPIHadoopRDD(...) // RDD containing spam information

    val cleanedDStream = wordCounts.transform(rdd => {
      rdd.join(spamInfoRDD).filter(...) // join data stream with spam information to do data cleaning
      ...
    })

#### Window操作 ####

window操作效果图如下图所示，把几个批次的DStream合并成一个DStream：

![](http://spark.apache.org/docs/latest/img/streaming-dstream-window.png)

每个window操作都需要2个参数：

1.window length。每个window对应的批次数(上图中是3，time1-time3是一个window, time3-time5也是一个window)
2.sliding interval。每个window之间的间隔时间，上图下方的window1，window3，window5的间隔。上图这个值为2

这两个参数必须是批次间隔的倍数。上个批次间隔值为1。

以1个例子来讲解window操作，基于本文一开始的那个例子，生成最后30秒的数据，每10秒为单位，这里就需要使用reduceByKeyAndWindow方法：

	val windowedWordCounts = pairs.reduceByKeyAndWindow((a:Int,b:Int) => (a + b), Seconds(30), Seconds(10))
    
其他的一些window操作：

| Transformation | 含义|
|:----:|:----:|
|  window(windowLength, slideInterval)   |  根据window操作的2个参数得到新的DStream  |
|  countByWindow(windowLength, slideInterval)   |  基于window操作的count操作  |
|  reduceByWindow(func, windowLength, slideInterval)   |  基于window操作的reduce操作  |
|  reduceByKeyAndWindow(func, windowLength, slideInterval, [numTasks])   |  基于window操作的reduceByKey操作  |
|  reduceByKeyAndWindow(func, invFunc, windowLength, slideInterval, [numTasks])   |  跟reduceByKeyAndWindow方法类似，更有效率，invFunc方法跟func方法的参数返回值一样，表示从window离开的数据  |
|  countByValueAndWindow(windowLength, slideInterval, [numTasks])   |  基于window操作的countByValue操作 |

#### Join操作 ####

DStream可以很容易地join其他DStream：

    val stream1: DStream[String, String] = ...
    val stream2: DStream[String, String] = ...
    val joinedStream = stream1.join(stream2)

还可以使用leftOuterJoin，rightOuterJoin，fullOuterJoin等。同样地，也可以在window操作后的DStream中使用join：

    val windowedStream1 = stream1.window(Seconds(20))
    val windowedStream2 = stream2.window(Minutes(1))
    val joinedStream = windowedStream1.join(windowedStream2)

基于rdd的join：

    val dataset: RDD[String, String] = ...
    val windowedStream = stream.window(Seconds(20))...
    val joinedStream = windowedStream.transform { rdd => rdd.join(dataset) }

### DStream的输出操作 ###

输出操作允许DStream中的数据输出到外部系统，比如像数据库、文件系统等。

| 输出操作 | 含义|
|:----:|:----:|
|  print()   |  打印出DStream中每个批次的前10条数据  |
|  saveAsTextFiles(prefix, [suffix])   |  把DStream中的数据保存到文本文件里。每次批次的文件名根据参数prefix和suffix生成："prefix-TIME_IN_MS[.suffix]"  |
|  saveAsObjectFiles(prefix, [suffix])   |  把DStream中的数据按照Java序列化的方式保存Sequence文件里，文件名规则跟saveAsTextFiles方法一样  |
|  saveAsHadoopFiles(prefix, [suffix])   |  把DStream中的数据保存到Hadoop文件里，文件名规则跟saveAsTextFiles方法一样  |
|  foreachRDD(func)   |  遍历DStream中的每段RDD，遍历的过程中可以将RDD中的数据保存到外部系统中  |

#### foreachRDD中的设计模式 ####

foreachRDD方法会遍历DStream中的每段RDD，遍历的过程中可以将RDD中的数据保存到外部系统中。这个方法很实用，所以理解foreachRDD方法显得很重要。

将数据写到外部系统通常都需要一个connection对象，所以很多时候都会不经意地创建这个connection对象：

    dstream.foreachRDD { rdd =>
      val connection = createNewConnection()  // executed at the driver
      rdd.foreach { record =>
        connection.send(record) // executed at the worker
      }
    }

这种写法是不正确的。这里connection需要被序列化并且发送到worker，而且connection对象会跨机器传递，会发生序列化错误(connection对象是不可序列化的)，初始化错误(connection对象需要在worker中初始化)。这个错误的解决方案就是在worker中创建connection对象。

为每条记录创建connection也是一个很常见的错误：

    dstream.foreachRDD { rdd =>
      rdd.foreach { record =>
        val connection = createNewConnection()
        connection.send(record)
        connection.close()
      }
    }

因为创建connection对象是一种很耗资源，很耗时间的操作。对于每条数据都创建一个connection代驾更大。所有可以使用rdd.foreachPartition方法，这个方法会创建单一的connection并且在一个RDD分区中所有数据都使用这个connection：

    dstream.foreachRDD { rdd =>
      rdd.foreachPartition { partitionOfRecords =>
        val connection = createNewConnection()
        partitionOfRecords.foreach(record => connection.send(record))
        connection.close()
      }
    }

一种更好的方式就是使用ConnectionPool，ConnectionPool可以重用connection对象在多个批次和RDD中。

    dstream.foreachRDD { rdd =>
      rdd.foreachPartition { partitionOfRecords =>
        // ConnectionPool is a static, lazily initialized pool of connections
        val connection = ConnectionPool.getConnection()
        partitionOfRecords.foreach(record => connection.send(record))
        ConnectionPool.returnConnection(connection)  // return to the pool for future reuse
      }
    }

其他需要注意的点：

1.DStream的输出操作也是延迟执行的，就像RDD的action操作一样。RDD的action操作在DStream的输出操作内部执行的话会强制Spark Streaming执行。如果程序里没有任何输出操作，或者有比如像dstream.foreachRDD操作一样内部没有rdd的action操作的话，这样就不会执行任意操作，会被Spark忽略。
2.默认情况下，在一个时间点下，只有一个输出操作被执行。它们是根据程序里的编写顺序执行的。
    

### DataFrame and SQL Operations ###

在Spark Streaming中可以使用DataFrames and SQL操作。
    
    /** DataFrame operations inside your streaming program */

    val words: DStream[String] = ...

    words.foreachRDD { rdd =>

      // Get the singleton instance of SQLContext
      val sqlContext = SQLContext.getOrCreate(rdd.sparkContext)
      import sqlContext.implicits._

      // Convert RDD[String] to DataFrame
      val wordsDataFrame = rdd.toDF("word")

      // Register as table
      wordsDataFrame.registerTempTable("words")

      // Do word count on DataFrame using SQL and print it
      val wordCountsDataFrame = 
        sqlContext.sql("select word, count(*) as total from words group by word")
      wordCountsDataFrame.show()
    }
    
### Caching / Persistence ###

跟RDD类似，DStream也允许将数据保存到内存中，使用persist方法可以做到这一点。

但是基于window和state的操作，reduceByWindow,reduceByKeyAndWindow,updateStateByKey它们就是隐式的保存了，系统已经帮它自动保存了。

从网络接收的数据(比如Kafka, Flume, sockets等)，默认是保存在两个节点来实现容错性，以序列化的方式保存在内存当中。
    
### Checkpointing ###
    
一个Spark Streaming程序必须是全天工作的，所以如果万一系统挂掉了或者JVM挂掉之后是要有容错性的。Spark Streaming需在容错存储系统做checkpoint，这样才能够处理错误信息。有两种类型的数据需要做checkpoint：

1.metadata checkpointing：元数据检查点。主要包括3个元数据：
配置：创建streaming程序的的配置信息
DStream操作：streaming程序中DStream的操作集合
未完成的批次：在队列中未完成的批次
2.data checkpointing：数据检查点。保存已经生成的RDD数据。在一些有状态的transformation操作中，一些RDD数据会依赖之前批次的RDD数据，随时时间的推移，这种依赖情况就会越发严重。为了解决这个问题，需要保存这些有依赖关系的RDD数据到存储系统中(比如HDFS)来剪断这种依赖关系

什么时候需要启用checkpoint？


满足以下2个条件中的任意1个即可启用checkpoint:

1.使用了有状态的transformation。比如使用了updateStateByKey或reduceByKeyAndWindow方法后，就需要启用checkpoint
2.恢复挂掉的程序。可以根据metadata数据恢复程序

一些比较简单的streaming程序没有用到有状态的transformation，并且也可以接受程序挂掉之后丢失部分数据，那么就没有必要启用checkpoint。


如何配置checkpoint？

checkpoint的配置需要设置一个目录，使用streamingContext.checkpoint(checkpointDirectory)方法。

    // Function to create and setup a new StreamingContext
    def functionToCreateContext(): StreamingContext = {
        val ssc = new StreamingContext(...)   // new context
        val lines = ssc.socketTextStream(...) // create DStreams
        ...
        ssc.checkpoint(checkpointDirectory)   // set checkpoint directory
        ssc
    }

    // Get StreamingContext from checkpoint data or create a new one
    val context = StreamingContext.getOrCreate(checkpointDirectory, functionToCreateContext _)

    // Do additional setup on context that needs to be done,
    // irrespective of whether it is being started or restarted
    context. ...

    // Start the context
    context.start()
    context.awaitTermination()

因为检查操作会导致保存到hdfs上的开销，所以设置这个时间间隔，要很慎重。对于小批次的数据，比如一秒的，检查操作会大大降低吞吐量。但是检查的间隔太长，会导致任务变大。通常来说，5-10秒的检查间隔时间是比较合适的。

    
