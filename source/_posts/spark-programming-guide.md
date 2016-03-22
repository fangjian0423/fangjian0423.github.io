title: Spark编程指南笔记
date: 2016-01-27 00:22:21
tags:
- big data
- spark
categories:
- spark
description: Spark编程指南笔记，参考官方文档的编程指南，翻译再加上一些自己写的代码 ...

--------------

## Spark初始化 ##

使用Spark编程第一件要做的事就是初始化SparkContext对象，SparkContext对象会告诉Spark如何使用Spark集群。

SparkContext会使用SparkConf中的一些配置信息，所以构造SparkContext对象之前需要构造一个SparkConf对象。

一个JVM上的SparkContext只有一个是激活的，如果要构造一个新的SparkContext，必须stop一个已经激活的SparkContext。

	val conf = new SparkConf().setMaster("local").setAppName("Test")

  val sc = new SparkContext(conf)
    
appName为Test，这个name会在cluster UI上展示，master是一个Spark，Mesos，YARN cluster URL或者local。 具体的值可以参考 [master-url解释](http://spark.apache.org/docs/latest/submitting-applications.html#master-urls)。

## RDD(Resilient Distributed Datasets) ##

Spark提出的最主要抽象概念是RDD(弹性分布式数据集)，它是一个有容错机制并且可以被并行操作的元素集合。

有两种方式可以创建RDD:

1.使用一个已存在的集合进行并行计算
2.使用外部数据集，比如共享的文件系统，HDFS，HBase以及任何支持Hadoop InputFormat的数据源

### 并行集合 ###

使用SparkContext的parallelize方法构造并行集合。

	val dataSet = Array(1,2,3,4,5)
    val rdd = sc.parallelize(dataSet)
    rdd.reduce(_ + _) // 15
    
parallelize方法有一个参数slices，表示数据集切分的份数。Spark会在集群上为每一个分片起一个任务。如果不设置的话，Spark会根据集群的情况，自动设置slices的数字。

### 外部数据集 ###

文本文件可以使用SparkContext的textFile方法构造RDD。这个方法接收一个URI参数(也可以包括本地的文件)，并且以每行的方式读取文件内容。

比如data.txt文件里一行一个数字，取所有数字的和：

	val rdd = sc.textFile(this.getClass.getResource("/data.txt").toString)

	rdd.reduce {
       (a, b) => (a.toInt + b.toInt).toString
    }
    
    // 另外一种方式
    rdd.map(s => s.toInt).reduce(_ + _)
    
Spark中所有基于文件的输入方法，都支持目录，压缩文件，通配符读取文件。比如

	sc.textFile("/data/*.txt")
    sc.textFile("/data")
    
textFile方法也可以传入第二个可选参数来控制文件的分片数量。默认情况下，Spark会为文件的每一个块（在HDFS中块的大小默认是64MB）创建一个分片。但是你也可以通过传入一个更大的值来要求Spark建立更多的分片。注意，分片的数量绝不能小于文件块的数量。

除了文本文件之外，Spark还支持其他格式的输入：

1.SparkContext的wholeTextFiles方法会读取一个包含很多小文件的目录，并以filename，content为键值对的方式返回结果。
2.对于SequenceFiles，可以使用SparkContext的sequenceFile[K, V]方法创建。像 IntWritable和Text一样，它们必须是 Hadoop 的 Writable 接口的子类。另外，对于几种通用 Writable 类型，Spark 允许你指定原生类型来替代。例如：sequencFile[Int, String] 将会自动读取 IntWritable 和 Texts。
3.对于其他类型的 Hadoop 输入格式，你可以使用 SparkContext.hadoopRDD 方法，它可以接收任意类型的 JobConf 和输入格式类，键类型和值类型。按照像 Hadoop 作业一样的方法设置输入源就可以了。
4.RDD.saveAsObjectFile 和 SparkContext.objectFile 提供了以 Java 序列化的简单方式来保存 RDD。虽然这种方式没有 Avro 高效，但也是一种简单的方式来保存任意的 RDD。


## RDD操作 ##

### RDD操作基础 ###

RDD支持两种类型的操作。

1.transformations。从一个数据集产生一个新的数据集。比如map方法，就可以根据旧的数据集产生新的数据集。
2.actions。在一个数据集中进行聚合操作，并且返回一个最终的结果。

Spark中所有的transformations操作都是lazy的，就是说它们并不会立刻真的计算出结果。相反，它们仅仅是记录下了转换操作的操作对象（比如：一个文件）。只有当一个启动操作被执行，要向驱动程序返回结果时，转化操作才会真的开始计算。这样的设计使得Spark运行更加高效——比如，我们会发觉由map操作产生的数据集将会在reduce操作中用到，之后仅仅是返回了reduce的最终的结果而不是map产生的庞大数据集。

在默认情况下，每一个由转化操作得到的RDD都会在每次执行启动操作时重新计算生成。但是，你也可以通过调用persist(或cache)方法来将RDD持久化到内存中，这样Spark就可以在下次使用这个数据集时快速获得。Spark同样提供了对将RDD持久化到硬盘上或在多个节点间复制的支持。

一个计算文件中每行的字符串个数和所有字符串个数的和例子：

	val rdd = sc.textFile("data.txt")
	val lineLengths = rdd.map(s => s.length)
	val totalLength = lineLengths.reduce(_ + _)
    
lineLengths对象是一个transformations结果，所以它不是马上就开始执行的，当运行lineLengths.reduce的时候lineLengths才会开始去计算。如果之后还会用到这个lineLengths。可以在reduce方法之前加上:

	lineLengths.persist()
    
### 使用函数 ###

Spark很多方法都可以使用函数完成。

使用对象：

	object SparkFunction {
      def strLength = (s: String) => s.length
    }

    val lineLengths = rdds.map(SparkFunction.strLength)
    lineLengths.reduce(_ + _)
    
使用类：

	class SparkCls {
      def func = (s: String) => s.length
      def buildRdd(rdd: RDD[String]) = rdd.map(func)
    }
	new SparkCls().buildRdd(rdds).reduce(_ + _)
    

### 闭包 ###

    var counter = 0
    var rdd = sc.parallelize(data)

    // Wrong: Don't do this!!
    rdd.foreach(x => counter += x)

    println("Counter value: " + counter)

上述代码如果在local模式并且在一个JVM的情况下使用是可以得到正确的值的。这是因为所有的RDD和变量counter都是同一块内存上。

然后在集群模式下，上述代码的结果可能就不会是我们想要的正确结果。集群模式下，Spark会在RDD分成多个任务，每个任务都会被对应的executor执行。在executor执行之前，Spark会计算每个闭包。上面这个例子foreach方法和counter就组成了一个闭包。这个闭包会被序列化并且发送给每个executor。在local模式下，因为只有一个executor，所以共享相同的闭包。然后在集群模式下，有多个executor，并且各个executor在不同的节点上都有自己的闭包的拷贝。

所以counter变量就已经不再是节点上的变量了。虽然counter变量在内存上依然存在，但是它对于executor已经不可见，executor只知道它是序列化后的闭包的一份拷贝。因此如果counter的操作都是在闭包下的话，counter的值还是为0。

Spark提供了一种Accumulator的概念用来处理集群模式下的变量更新问题。


另外一个要注意的是不要使用foreach或者map方法打印数据。在一台机器上，这个操作是没有问题的。但是如果在集群上，不一定会打印出全部的数据。可以使用collect方法将RDD放到调用节点上。所以rdd.collect().foreach(println)是可以打印出数据的，但是可能数据量过大，会导致OOM。所以最好的方式还是使用take方法：rdd.take(100).foreach(println)。

### 使用键值对 ###

Spark也支持键值对的操作，这在分组和聚合操作时候用得到。当键值对中的键为自定义对象时，需要自定义该对象的equals()和hashCode()方法。

一个使用键值对的单词统计例子：

	// 使用map方法将单词文本转换成一个键值对，(word, num)。 num初始值为1
	val pairs = rdd.map(s => (s, 1))
    val reduceRdd = pairs.reduceByKey(_ + _)
    val result = reduceRdd.sortByKey().collect()
    result.foreach(println)

### Spark内置的Transformations ###

| 转换 | 含义|
|:----:|:----:|
|  map(func)   |  根据func函数生成一个新的rdd数据集  |
|  filter(func)   |  根据func函数返回true的数据集   |
|  flatMap(func)   |  跟map方法类似，但是每一项可以返回多个值。func函数的返回值是一个集合   |
|  mapPartitions(func)   |  跟map方法类似，但是是在每个partition上运行的。func函数的参数是一个Iteraror，返回值也是一个Iterator。如果map方法需要创建一个额外的对象，使用mapPartitions方法比map方法高效得多   |
|  mapPartitionsWithIndex(func)   |  作用跟mapPartitions方法一样，只是func方法多了一个index参数。 func方法定义 (Int, Iterator[T]) => Iterator[U]   |
|  sample(withReplacement, fraction, seed)   |  根据 fraction 指定的比例，对数据进行采样，可以选择是否用随机数进行替换，seed 用于指定随机数生成器种子   |
|  union(otherDataset)   |  取2个rdd的并集，得到一个新的rdd   |
|  intersection(otherDataset)   |  取2个rdd的交集，得到一个新的rdd。这个新的rdd没有重复的数据   |
|  distinct([numTasks])   |  返回一个新的没有重复数据的数据集   |
|  groupByKey([numTasks])   |  将一个(K,V)的键值对RDD转换成一个(K, Iterable[V])的新的键值对RDD。注意点：如果group的目的是为了做聚合计算(比如总和或者平均值)，使用reduceByKey或者aggregateByKey性能更好。 |
|  reduceByKey(func, [numTasks])   |  跟groupByKey方法一样，也是操作(K, V)的键值对RDD。返回值同样是一个(K, V)的键值对RDD，func函数的定义：(V, V) => V，也就是每两个值的值  |
|  aggregateByKey(zeroValue)(seqOp, combOp, [numTasks])	   |  跟reduceByKey作用类似，zeroValue参数表示初始值，这个初始值的类型可以跟rdd中的键值对的值的类型不同。seqOp参数是个函数，定义为(U, V) => U，U类型是初始化zeroValue的类型，V类型是一开始rdd的键值对的值的类型。这个函数表示用来与初始值zeroValue进行比较，取一个新的值，需要注意的是这个新的值会作为参数出现在下一次key相等的情况下。 combOp参数也是个函数，定义为(U, U) => U，U类型也是初始值的类型。这个函数相当于reduce方法中的函数，用来做聚合操作    |
|  sortByKey([ascending], [numTasks])   |  对一个(K, V)键值对的RDD进行排行，返回一个基于K排序的新的RDD  |
|  join(otherDataset, [numTasks])   |  基于(K, V)键值对的rdd，如果对(K, W)的键值对rdd使用join操作，可以产生(K, (V, W))键值对的rdd。类似数据库中的join操作，spark还提供leftOuterJoin, rightOuterJoin, fullOuterJoin方法  |
|  cogroup(otherDataset, [numTasks])   |   跟join方法类似，不过是基于(K, V)的rdd，cogroup基于(K, W)的rdd，产生(K, (Iterable[V], Iterable[W]))的rdd。这个方法也叫做groupWith  |
|  cartesian(otherDataset)   |  笛卡尔积。 有K的rdd与V的rdd进行笛卡尔积，会生成(K, V)的rdd   |
|  pipe(command, [envVars])   |  对rdd进行管道操作。 就像shell命令一样   |
|  coalesce(numPartitions)   |  减少 RDD 的分区数到指定值。在过滤大量数据之后，可以执行此操作   |
|  repartition(numPartitions)   |  	重新给 RDD 分区   |
|  repartitionAndSortWithinPartitions(partitioner)	   |  	重新给 RDD 分区，并且每个分区内以记录的 key 排序   |

以这个数据为例：

	i
    am
    format
    let
    us
    go
    hoho
    good
    nice
    format
    is
    nice
    haha
    haha
    haha
    scala is cool, nice

一些Transformations操作：

	rdd.map(s => s.length) // 每一行文本转换成长度
    rdd.filter(s => s.length == 1) // 取文本长度为1的数据
    rdd.flatMap(s => s.split(",")) // 把有 , 的字符串转换成多行
    rdd.sample(false, 1.0)
    rdd.union(rdd2)
    rdd.intersection(rdd2)
	rdd.distinct(rdd2)
    rdd.map(s => (s, 1)).groupByKey().foreach {
      pair => {
        print(pair._1) ; print(" ** ")
        println(pair._2.mkString("-"))
      }
    }
    rdd.map(s => (s, 1)).reduceByKey {
      (x, y) => x + y
    }.foreach {
      (pair) => println(pair._1 + " " + pair._2)
    }
    
    rdd.map(s => (s, 1)).aggregateByKey(0)(
      (a, b) => {
        math.max(a, b)
      }, (a, b) => {
        a + b
      }
    ).foreach {
      pair => println(pair._1 + " " + pair._2)
    }
    
    rdd.map(s => (s, 1)).sortByKey(true).foreach {
      pair => println(pair._1 + " " + pair._2)
    }
    
    rdd.map(s => (s, s.length)).join(rdd.map(s => (s, s.charAt(0).toUpper.toString))).foreach {
      pair => println(pair._1 + " " + pair._2._1 + " " + pair._2._2)
    }
    
    rdd.map(s => (s, s.length)).cogroup(rdd.map(s => (s, s.charAt(0).toUpper.toString))).foreach {
      pair => {
        println(pair._1 + "======")
        println(pair._2._1.toList.mkString("-"))
        println(pair._2._2.toList.mkString("-"))
        println("**" * 8)
      }
    }
    
    rdd.map(s => s.length).cartesian(rdd).foreach {
      pair => {
        println(pair._1)
        println(pair._2)
        println("**" * 8)
      }
    }
    
### Spark内置的Actions ###

| 动作 | 含义|
|:----:|:----:|
|  reduce(func)   |  聚合操作。之前很多例子都使用了reduce方法。这个功能必须可交换且可关联的，从而可以正确的被并行执行。   |
|  collect()   |  返回rdd中所有的元素，返回值类型是Array。这个方法经常用来取数据量比较小的集合   |
|  count()   |  rdd中的元素个数   |
|  first()   |  返回数据集的第一个元素   |
|  take(n)   |  返回数据集的前n个元素，返回值是个Array   |
|  takeSample(withReplacement, num, [seed])   |  返回一个数组，在数据集中随机采样 num 个元素组成，可以选择是否用随机数替换不足的部分，seed 用于指定的随机数生成器种子返回数据集的前n个元素，返回值是个Array   |
|  takeOrdered(n, [ordering])   |  返回自然顺序或者自定义顺序的前 n 个元素   |
|  saveAsTextFile(path)   |  把数据集中的元素写到文件里，可以写到本地文件系统上，hdfs上或者任意Hadoop支持的文件系统上。Spark会调用元素的toString方法将其转换成文本的一行   |
|  saveAsSequenceFile(path)   |  跟saveAsTextFile方法类似，但是是写成SequenceFile文件格式，也是支持写到本地文件系统上，hdfs上或者任意Hadoop支持的文件系统上。这个方法只能作用于键值对的RDD   |
|  saveAsObjectFile(path)    |  跟saveAsTextFile方法类似，是使用Java的序列化的方式保存文件   |
|  countByKey()   |  计算键值的数量。对键值对(K, V)的rdd数据集，返回(K, Int)的Map   |
|  foreach(func)   |  使用func遍历rdd数据集中的各个元素。这通常用于边缘效果，例如更新一个Accumulator，或者和外部存储系统进行交互   |

一些Actions操作：

	rdd.saveAsTextFile("file:///tmp/data01")
    rdd.map(s => (s, s.length)).saveAsSequenceFile("file:///tmp/data02")
    rdd.map(s => (s, s.length)).countByKey().foreach {
      pair => println(pair._1 + " " + pair._2)
    }
    

## RDD的持久化(Persistence) ##

Spark的一个重要功能就是在将数据集持久化（或缓存）到内存中以便在多个操作中重复使用。当持久化一个RDD的时候，每个存储着这个RDD的分片节点都会计算然后保存到内存中以便下次再次使用。这使得接下来的计算过程速度能够加快（经常能加快超过十倍的速度）。缓存是加快迭代算法和快速交互过程速度的关键工具。

可以使用persist或者cache方法让rdd持久化。在第一次被计算产生之后，它就会始终停留在节点的内存中。Spark的缓存是具有容错性的——如果RDD的任意一个分片丢失了，Spark就会依照这个RDD产生的转化过程自动重算一遍。

另外，每个持久化后的RDD可以使用不用级别的存储级别。比如可以存在硬盘中，可以存在内存中，还可以将这个数据集在节点之间复制，或者使用 Tachyon 将它储存到堆外。这些存储级别都是通过向 persist() 传递一个 StorageLevel 对象（Scala, Java, Python）来设置的。

Spark的一些存储级别如下：


|  存储级别  | 含义 |
|:----:|:----:|
|  MEMORY_ONLY   |  默认级别。将RDD作为反序列化的的对象存储在JVM中。如果不能被内存装下，一些分区将不会被缓存，并且在需要的时候被重新计算   |
|  MEMORY_AND_DISK   |  默认级别。将RDD作为反序列化的的对象存储在JVM中。如果不能被内存装下，会存在硬盘上，并且在需要的时候被重新计算   |
|  MEMORY_ONLY_SER   |  将RDD作为序列化的的对象进行存储（每一分区占用一个字节数组）。通常来说，这比将对象反序列化的空间利用率更高，尤其当使用fast serializer,但在读取时会比较占用CPU   |
|  MEMORY_AND_DISK_SER   |  与MEMORY_ONLY_SER相似，但是把超出内存的分区将存储在硬盘上而不是在每次需要的时候重新计算   |
|  DISK_ONLY   |  只存储RDD分区到硬盘上   |
|  MEMORY_ONLY_2, MEMORY_AND_DISK_2 等	   |  与上述的存储级别一样，但是将每一个分区都复制到两个集群结点上   |


存储级别的选择：

如果你的 RDD 可以很好的与默认的存储级别契合，就不需要做任何修改了。这已经是 CPU 使用效率最高的选项，它使得 RDD的操作尽可能的快。

如果不行，试着使用 MEMORY_ONLY_SER 并且选择一个快速序列化的库使得对象在有比较高的空间使用率的情况下，依然可以较快被访问。

尽可能不要存储到硬盘上，除非计算数据集的函数，计算量特别大，或者它们过滤了大量的数据。否则，重新计算一个分区的速度，和与从硬盘中读取基本差不多快。

如果你想有快速故障恢复能力，使用复制存储级别。例如：用 Spark 来响应web应用的请求。所有的存储级别都有通过重新计算丢失数据恢复错误的容错机制，但是复制存储级别可以让你在 RDD 上持续的运行任务，而不需要等待丢失的分区被重新计算。

如果你想要定义你自己的存储级别，比如复制因子为3而不是2，可以使用 StorageLevel 单例对象的 apply()方法。

## 共享变量 ##

通常情况下，当一个函数在远程集群节点上通过Spark操作(比如map或者reduce)，Spark会对涉及到的变量的所有副本执行这个函数。这些变量都会被拷贝到每台机器上，而且这个过程不会被反馈到驱动程序。通常情况下，在任务之间读写共享变量是很低效的。但是Spark仍然提供了有限的两种共享变量类型用于常见的使用场景：broadcast variables 和 accumulators。


### broadcast variables(广播变量) ###

广播变量允许程序员在每台机器上保持一个只读变量的缓存而不是将一个变量的拷贝传递给各个任务。这些变量是可以被使用的，比如，给每个节点传递一份大输入数据集的拷贝是很耗时的。Spark试图使用高效的广播算法来分布广播变量，以此来降低通信花销。可以通过SparkContext.broadcast(v)来从变量v创建一个广播变量。这个广播变量是v的一个包装，同时它的值可以调用value方法获得：

	val broadcastVar = sc.broadcast(Array(1, 2, 3))
    
    broadcastVar.value // Array(1, 2, 3)
    
一个广播变量被创建之后，在所有函数中都应当使用它来代替原来的变量v，这样就可以包装v在节点之间只被传递一次。另外，v变量在被广播之后不应该再被修改，这样可以确保每一个节点上储存的广播变量的一致性。

### accumulators(累加器) ###

累加器是在一个相关过程中只能被”累加”的变量，对这个变量的操作可以有效地被并行化。它们可以被用于实现计数器（就像在MapReduce过程中）或求和运算。Spark原生支持对数字类型的累加器，程序员也可以为其他新的类型添加支持。累加器被以一个名字创建之后，会在Spark的UI中显示出来。这有助于了解计算的累进过程（注意：目前Python中不支持这个特性）。


可以通过SparkContext.accumulator(v)来从变量v创建一个累加器。在集群中运行的任务随后可以使用add方法或+=操作符（在Scala和Python中）来向这个累加器中累加值。但是，他们不能读取累加器中的值。只有驱动程序可以读取累加器中的值，通过累加器的value方法。


以下的代码展示了向一个累加器中累加数组元素的过程：

	val accum = sc.accumulator(0, "My Accumulator")
    sc.parallelize(Array(1, 2, 3, 4)).foreach(x => accum += x)
    accum.value // 10
    
这段代码利用了累加器对Int类型的内建支持，程序员可以通过继承 AccumulatorParam 类来创建自己想要的类型支持。AccumulatorParam 的接口提供了两个方法：zero 用于为你的数据类型提供零值；addInPlace 用于计算两个值得和。比如，假设我们有一个 Vector类表示数学中的向量，我们可以这样写：

    object VectorAccumulatorParam extends AccumulatorParam[Vector] {
      def zero(initialValue: Vector): Vector = {
        Vector.zeros(initialValue.size)
      }
      def addInPlace(v1: Vector, v2: Vector): Vector = {
        v1 += v2
      }
    }

	// Then, create an Accumulator of this type:
	val vecAccum = sc.accumulator(new Vector(...))(VectorAccumulatorParam)
    
累加器的更新操作只会被运行一次，Spark 提供了保证，每个任务中对累加器的更新操作都只会被运行一次。比如，重启一个任务不会再次更新累加器。在转化过程中，用户应该留意每个任务的更新操作在任务或作业重新运算时是否被执行了超过一次。

累加器不会改变Spark的惰性求值模型。如果累加器在对RDD的操作中被更新了，它们的值只会在启动操作中作为 RDD 计算过程中的一部分被更新。所以，在一个懒惰的转化操作中调用累加器的更新，并没法保证会被及时运行。下面的代码段展示了这一点：

    accum = sc.accumulator(0)
    data.map(lambda x => acc.add(x); f(x))
    
## 参考资料 ##

[http://spark.apache.org/docs/latest/programming-guide.html](http://spark.apache.org/docs/latest/programming-guide.html/)
    
