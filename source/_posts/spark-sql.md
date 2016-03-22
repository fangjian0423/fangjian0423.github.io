title: Spark DataFrame介绍
date: 2016-02-17 09:22:22
tags:
- spark
- big data
categories:
- spark
description: DataFrame是一个以命名列方式组织的分布式数据集。在概念上，它跟关系型数据库中的一张表或者1个Python(或者R)中的data frame一样，但是比他们更优化。DataFrame可以根据结构化的数据文件、hive表、外部数据库或者已经存在的RDD构造 ...

--------------

## DataFrame是什么 ##

DataFrame是一个以命名列方式组织的分布式数据集。在概念上，它跟关系型数据库中的一张表或者1个Python(或者R)中的data frame一样，但是比他们更优化。DataFrame可以根据结构化的数据文件、hive表、外部数据库或者已经存在的RDD构造。

## DataFrame的创建 ##

Spark DataFrame可以从一个已经存在的RDD、hive表或者数据源中创建。

以下一个例子就表示一个DataFrame基于一个json文件创建：

    val sc: SparkContext // An existing SparkContext.
    val sqlContext = new org.apache.spark.sql.SQLContext(sc)

    val df = sqlContext.read.json("examples/src/main/resources/people.json")

    // Displays the content of the DataFrame to stdout
    df.show()
    
## DataFrame的操作 ##

直接以1个例子来说明DataFrame的操作：

json文件内容：

    {"name":"Michael"}
    {"name":"Andy", "age":30}
    {"name":"Justin", "age":19}
    
程序内容：
    
    val conf = new SparkConf().setMaster("local").setAppName("DataFrameTest")
  
    val sc = new SparkContext(conf)
  
    val sqlContext = new SQLContext(sc)
  
    val df = sqlContext.read.json(this.getClass.getResource("/").toString + "people.json")
  
  	/** 展示DataFrame的内容
    +----+-------+
    | age|   name|
    +----+-------+
    |null|Michael|
    |  30|   Andy|
    |  19| Justin|
    +----+-------+  
  	**/
    df.show()
    
    /** 以树的形式打印出DataFrame的schema
    root
     |-- age: long (nullable = true)
     |-- name: string (nullable = true)
    **/
    df.printSchema()
    
    /** 打印出name列的数据
    +-------+
    |   name|
    +-------+
    |Michael|
    |   Andy|
    | Justin|
    +-------+   
    **/
    df.select("name").show()
    
    /** 打印出name列和age列+1的数据，DataFrame的apply方法返回Column
    +-------+---------+
    |   name|(age + 1)|
    +-------+---------+
    |Michael|     null|
    |   Andy|       31|
    | Justin|       20|
    +-------+---------+
    **/
    df.select(df("name"), df("age") + 1).show()
    
    /** 添加过滤条件，过滤出age字段大于21的数据
    +---+----+
    |age|name|
    +---+----+
    | 30|Andy|
    +---+----+
    **/
    df.filter(df("age") > 21).show()
    
    /** 以age字段分组进行统计
    +----+-----+
    | age|count|
    +----+-----+
    |null|    1|
    |  19|    1|
    |  30|    1|
    +----+-----+
    **/
    df.groupBy(df("age")).count().show()
    
## 使用反射推断出Schema ##

Spark SQL的Scala接口支持将包括case class数据的RDD转换成DataFrame。

case class定义表的schema，case class的属性会被读取并且成为列的名字，这里case class也可以被当成别的case class的属性或者是复杂的类型，比如Sequence或Array。

RDD会被隐式转换成DataFrame并且被注册成一个表，这个表可以被用在查询语句中：

    // sc is an existing SparkContext.
    val sqlContext = new org.apache.spark.sql.SQLContext(sc)
    // this is used to implicitly convert an RDD to a DataFrame.
    import sqlContext.implicits._

    // Define the schema using a case class.
    // Note: Case classes in Scala 2.10 can support only up to 22 fields. To work around this limit,
    // you can use custom classes that implement the Product interface.
    case class Person(name: String, age: Int)

    // Create an RDD of Person objects and register it as a table.
    val people = sc.textFile("examples/src/main/resources/people.txt").map(_.split(",")).map(p => Person(p(0), p(1).trim.toInt)).toDF()
    people.registerTempTable("people")

    // SQL statements can be run by using the sql methods provided by sqlContext.
    val teenagers = sqlContext.sql("SELECT name, age FROM people WHERE age >= 13 AND age <= 19")

    // The results of SQL queries are DataFrames and support all the normal RDD operations.
    // The columns of a row in the result can be accessed by field index:
    teenagers.map(t => "Name: " + t(0)).collect().foreach(println)

    // or by field name:
    teenagers.map(t => "Name: " + t.getAs[String]("name")).collect().foreach(println)

    // row.getValuesMap[T] retrieves multiple columns at once into a Map[String, T]
    teenagers.map(_.getValuesMap[Any](List("name", "age"))).collect().foreach(println)
    // Map("name" -> "Justin", "age" -> 19)

## 使用编程指定Schema ##

当case class不能提前确定（例如，记录的结构是经过编码的字符串，或者一个文本集合将会被解析，不同的字段投影给不同的用户），一个 DataFrame 可以通过三步来创建。

1.从原来的 RDD 创建一个行的 RDD
2.创建由一个 StructType 表示的模式与第一步创建的 RDD 的行结构相匹配
3.在行 RDD 上通过 applySchema 方法应用模式

    // sc is an existing SparkContext.
    val sqlContext = new org.apache.spark.sql.SQLContext(sc)

    // Create an RDD
    val people = sc.textFile("examples/src/main/resources/people.txt")

    // The schema is encoded in a string
    val schemaString = "name age"

    // Import Row.
    import org.apache.spark.sql.Row;

    // Import Spark SQL data types
    import org.apache.spark.sql.types.{StructType,StructField,StringType};

    // Generate the schema based on the string of schema
    val schema =
      StructType(
        schemaString.split(" ").map(fieldName => StructField(fieldName, StringType, true)))

    // Convert records of the RDD (people) to Rows.
    val rowRDD = people.map(_.split(",")).map(p => Row(p(0), p(1).trim))

    // Apply the schema to the RDD.
    val peopleDataFrame = sqlContext.createDataFrame(rowRDD, schema)

    // Register the DataFrames as a table.
    peopleDataFrame.registerTempTable("people")

    // SQL statements can be run by using the sql methods provided by sqlContext.
    val results = sqlContext.sql("SELECT name FROM people")

    // The results of SQL queries are DataFrames and support all the normal RDD operations.
    // The columns of a row in the result can be accessed by field index or by field name.
    results.map(t => "Name: " + t(0)).collect().foreach(println)
    
## 数据源 ##

Spark SQL默认使用的数据源是parquet(可以通过spark.sql.sources.default修改)。

	val df = sqlContext.read.load("examples/src/main/resources/users.parquet")
	df.select("name", "favorite_color").write.save("namesAndFavColors.parquet")

可以在读取数据源的时候指定一些往外的参数。数据源也可以使用全名称，比如org.apache.spark.sql.parquet，但是内置的数据源可以使用短名称，比如json, parquet, jdbc。任何类型的DataFrame都可以使用这种方式转换成其他类型：

    val df = sqlContext.read.format("json").load("examples/src/main/resources/people.json")
    df.select("name", "age").write.format("parquet").save("namesAndAges.parquet")


使用read方法读取数据源得到DataFrame，还可以使用sql直接查询文件的方式：

	val df = sqlContext.sql("SELECT * FROM parquet.`examples/src/main/resources/users.parquet`")

保存模式：

保存方法会需要一个可选参数SaveMode，用于处理已经存在的数据。这些保存模式内部不会用到锁的概念，也不是一个原子操作。如果使用了Overwrite这种保存模式，那么写入数据前会清空之前的老数据。

| Scala/Java | 具体值| 含义 |
|:----:|:----:|:----:|
|  SaveMode.ErrorIfExists (默认值)   |  "error" (默认值)  | 当保存DataFrame到数据源的时候，如果数据源文件已经存在，那么会抛出异常 |
|  SaveMode.Append   |  "append"  | 如果数据源文件已经存在，append到文件末尾 |
|  SaveMode.Overwrite   |  "overwrite"  | 如果数据源文件已经存在，清空数据 |
|  SaveMode.Ignore   |  "ignore"  | 如果数据源文件已经存在，不做任何处理。跟SQL中的 CREATE TABLE IF NOT EXISTS 类似 |


持久化表：

当使用HiveContext的时候，使用saveAsTable方法可以把DataFrame持久化成表。跟registerTempTable方法不一样，saveAsTable方法会把DataFrame持久化成表，并且创建一个数据的指针到HiveMetastore对象中。只要获得了同一个HiveMetastore对象的链接，当Spark程序重启的时候，saveAsTable持久化后的表依然会存在。一个DataFrame持久化成一个table也可以通过SQLContext的table方法，参数就是表的名字。

默认情况下，saveAsTable方法会创建一个"被管理的表"，被管理的表的意思是说表中数据的位置会被HiveMetastore所控制，如果表被删除了，HiveMetastore中的数据也相当于被删除了。

### Parquet Files ###

parquet是一种基于列的存储格式，并且可以被很多框架所支持。Spark SQL支持parquet文件的读和写操作，并且会自动维护原始数据的schema，当写一个parquet文件的时候，所有的列都允许为空。

#### 加载Parquet文件 ####

    // sqlContext from the previous example is used in this example.
    // This is used to implicitly convert an RDD to a DataFrame.
    import sqlContext.implicits._

    val people: RDD[Person] = ... // An RDD of case class objects, from the previous example.

    // The RDD is implicitly converted to a DataFrame by implicits, allowing it to be stored using Parquet.
    people.write.parquet("people.parquet")

    // Read in the parquet file created above. Parquet files are self-describing so the schema is preserved.
    // The result of loading a Parquet file is also a DataFrame.
    val parquetFile = sqlContext.read.parquet("people.parquet")

    //Parquet files can also be registered as tables and then used in SQL statements.
    parquetFile.registerTempTable("parquetFile")
    val teenagers = sqlContext.sql("SELECT name FROM parquetFile WHERE age >= 13 AND age <= 19")
    teenagers.map(t => "Name: " + t(0)).collect().foreach(println)

#### Parquet文件的Partition ####

Parquet文件可以根据列自动进行分区，只需要调用DataFrameWriter的partitionBy方法即可，该方法需要的参数是需要进行分区的列。比如需要分区成这样：

	path
    └── to
        └── table
            ├── gender=male
            │   ├── ...
            │   │
            │   ├── country=US
            │   │   └── data.parquet
            │   ├── country=CN
            │   │   └── data.parquet
            │   └── ...
            └── gender=female
                ├── ...
                │
                ├── country=US
                │   └── data.parquet
                ├── country=CN
                │   └── data.parquet
                └── ...

这个需要DataFrame就需要4列，分别是name，age，gender和country，write的时候如下：

	dataFrame.write.partitionBy("gender", "country").parquet("path")

#### Schema Merging ####

像ProtocolBuffer，Avro，Thrift一样，Parquet也支持schema的扩展。

由于schema的自动扩展是一次昂贵的操作，所以默认情况下不是开启的，可以根据以下设置打开：

读parquet文件的时候设置参数mergeSchema为true或者设置全局的sql属性spark.sql.parquet.mergeSchema为true：

    // sqlContext from the previous example is used in this example.
    // This is used to implicitly convert an RDD to a DataFrame.
    import sqlContext.implicits._

    // Create a simple DataFrame, stored into a partition directory
    val df1 = sc.makeRDD(1 to 5).map(i => (i, i * 2)).toDF("single", "double")
    df1.write.parquet("data/test_table/key=1")

    // Create another DataFrame in a new partition directory,
    // adding a new column and dropping an existing column
    val df2 = sc.makeRDD(6 to 10).map(i => (i, i * 3)).toDF("single", "triple")
    df2.write.parquet("data/test_table/key=2")

    // Read the partitioned table
    val df3 = sqlContext.read.option("mergeSchema", "true").parquet("data/test_table")
    df3.printSchema()

    // The final schema consists of all 3 columns in the Parquet files together
    // with the partitioning column appeared in the partition directory paths.
    // root
    // |-- single: int (nullable = true)
    // |-- double: int (nullable = true)
    // |-- triple: int (nullable = true)
    // |-- key : int (nullable = true)

### JSON数据源 ###

本文之前的一个例子就是使用的JSON数据源，使用SQLContext.read.json()读取一个带有String类型的RDD或者一个json文件。

需要注意的是json文件不是一个典型的json格式的文件，每一行都是一个json对象。

    // sc is an existing SparkContext.
    val sqlContext = new org.apache.spark.sql.SQLContext(sc)

    // A JSON dataset is pointed to by path.
    // The path can be either a single text file or a directory storing text files.
    val path = "examples/src/main/resources/people.json"
    val people = sqlContext.read.json(path)

    // The inferred schema can be visualized using the printSchema() method.
    people.printSchema()
    // root
    //  |-- age: integer (nullable = true)
    //  |-- name: string (nullable = true)

    // Register this DataFrame as a table.
    people.registerTempTable("people")

    // SQL statements can be run by using the sql methods provided by sqlContext.
    val teenagers = sqlContext.sql("SELECT name FROM people WHERE age >= 13 AND age <= 19")

    // Alternatively, a DataFrame can be created for a JSON dataset represented by
    // an RDD[String] storing one JSON object per string.
    val anotherPeopleRDD = sc.parallelize(
      """{"name":"Yin","address":{"city":"Columbus","state":"Ohio"}}""" :: Nil)
    val anotherPeople = sqlContext.read.json(anotherPeopleRDD)

### Hive表 ###

需要使用HiveContext。

    // sc is an existing SparkContext.
    val sqlContext = new org.apache.spark.sql.hive.HiveContext(sc)

    sqlContext.sql("CREATE TABLE IF NOT EXISTS src (key INT, value STRING)")
    sqlContext.sql("LOAD DATA LOCAL INPATH 'examples/src/main/resources/kv1.txt' INTO TABLE src")

    // Queries are expressed in HiveQL
    sqlContext.sql("FROM src SELECT key, value").collect().foreach(println)

### JDBC ###

直接使用load方法加载：

	sqlContext.load("jdbc", Map("url" -> "jdbc:mysql://localhost:3306/your_database?user=your_user&password=your_password", "dbtable" -> "your_table"))
