title: Avro介绍
date: 2016-02-21 01:23:22
tags:
- big data
- avro
categories:
- avro
description: Apache Avro是一个数据序列化系统, 提供丰富的数据结构，使用快速的压缩二进制数据格式，提供容器文件用于持久化数据 ...

--------------

## Avro介绍 ##

Apache Avro是一个数据序列化系统。

Avro所提供的属性：

1.丰富的数据结构
2.使用快速的压缩二进制数据格式
3.提供容器文件用于持久化数据
4.远程过程调用RPC
5.简单的动态语言结合功能，Avro 和动态语言结合后，读写数据文件和使用 RPC 协议都不需要生成代码，而代码生成作为一种可选的优化只值得在静态类型语言中实现。


## Avro的Schema ##

Avro的Schema用JSON表示。Schema定义了简单数据类型和复杂数据类型。

### 基本类型 ###

其中简单数据类型有以下8种：

| 类型 | 含义 |
|:----:|:----:|
|  null   |  没有值  |
|  boolean   |  布尔值  |
|  int   |  32位有符号整数  |
|  long   |  64位有符号整数  |
|  float   |  单精度（32位）的IEEE 754浮点数  |
|  double   |  双精度（64位）的IEEE 754浮点数  |
|  bytes   |  8位无符号字节序列  |
|  string   |  字符串  |

基本类型没有属性，基本类型的名字也就是类型的名字，比如：

	{"type": "string"}
    
### 复杂类型 ###

Avro提供了6种复杂类型。分别是Record，Enum，Array，Map，Union和Fixed。

#### Record ####

Record类型使用的类型名字是 "record"，还支持其它属性的设置：

name：record类型的名字(必填)

namespace：命名空间(可选)

doc：这个类型的文档说明(可选)

aliases：record类型的别名，是个字符串数组(可选)

fields：record类型中的字段，是个对象数组(必填)。每个字段需要以下属性：

1. name：字段名字(必填)
2. doc：字段说明文档(可选)
3. type：一个schema的json对象或者一个类型名字(必填)
4. default：默认值(可选)
5. order：排序(可选)，只有3个值ascending(默认)，descending或ignore
6. aliases：别名，字符串数组(可选)

一个Record类型例子，定义一个元素类型是Long的链表：

    {
      "type": "record", 
      "name": "LongList",
      "aliases": ["LinkedLongs"],                      // old name for this
      "fields" : [
        {"name": "value", "type": "long"},             // each element has a long
        {"name": "next", "type": ["null", "LongList"]} // optional next element
      ]
    }

#### Enum ####

枚举类型的类型名字是"enum"，还支持其它属性的设置：

name：枚举类型的名字(必填)
namespace：命名空间(可选)
aliases：字符串数组，别名(可选)
doc：说明文档(可选)
symbols：字符串数组，所有的枚举值(必填)，不允许重复数据。

一个枚举类型的例子：

    { "type": "enum",
      "name": "Suit",
      "symbols" : ["SPADES", "HEARTS", "DIAMONDS", "CLUBS"]
    }

#### Array ####

数组类型的类型名字是"array"并且只支持一个属性：

items：数组元素的schema

一个数组例子：

	{"type": "array", "items": "string"}
    
#### Map ####

Map类型的类型名字是"map"并且只支持一个属性：

values：map值的schema

Map的key必须是字符串。

一个Map例子：

	{"type": "map", "values": "long"}

#### Union ####

组合类型，表示各种类型的组合，使用数组进行组合。比如["null", "string"]表示类型可以为null或者string。

组合类型的默认值是看组合类型的第一个元素，因此如果一个组合类型包括null类型，那么null类型一般都会放在第一个位置，这样子的话这个组合类型的默认值就是null。

组合类型中不允许同一种类型的元素的个数不会超过1个，除了record，fixed和enum。比如组合类中有2个array类型或者2个map类型，这是不允许的。

组合类型不允许嵌套组合类型。

#### Fixed ####

混合类型的类型名字是fixed，支持以下属性：

name：名字(必填)
namespace：命名空间(可选)
aliases：字符串数组，别名(可选)
size：一个整数，表示每个值的字节数(必填)

比如16个字节数的fixed类型例子如下：

	{"type": "fixed", "size": 16, "name": "md5"}

## 1个Avro例子 ##

首先定义一个User的schema：

    {
    "namespace": "example.avro",
     "type": "record",
     "name": "User",
     "fields": [
         {"name": "name", "type": "string"},
         {"name": "favorite_number",  "type": "int"},
         {"name": "favorite_color", "type": "string"}
     ]
    }

User有3个属性，分别是name，favorite_number和favorite_color。

json文件内容：

    {"name":"format","favorite_number":1,"favorite_color":"red"}
    {"name":"format2","favorite_number":2,"favorite_color":"black"}
	{"name":"format3","favorite_number":666,"favorite_color":"blue"}

使用avro工具将json文件转换成avro文件：

	java -jar avro-tools-1.8.0.jar fromjson --schema-file user.avsc user.json > user.avro
    
可以设置压缩格式：

	java -jar avro-tools-1.8.0.jar fromjson --codec snappy --schema-file user.avsc user.json > user2.avro
    
将avro文件反转换成json文件：

	java -jar avro-tools-1.8.0.jar tojson user.avro
    java -jar avro-tools-1.8.0.jar --pretty tojson user.avro
    
得到avro文件的meta：

	java -jar avro-tools-1.8.0.jar getmeta user.avro

输出：

	avro.codec	null
    avro.schema	{"type":"record","name":"User","namespace":"example.avro","fields":[{"name":"name","type":"string"},{"name":"favorite_number","type":"int"},{"name":"favorite_color","type":"string"}]}

得到avro文件的schema：

	java -jar avro-tools-1.8.0.jar getschema user.avro
    
将文本文件转换成avro文件：

	java -jar avro-tools-1.8.0.jar fromtext user.txt usertxt.avro
    
## Avro使用生成的代码进行序列化和反序列化 ##

以上面一个例子的schema为例讲解。

Avro可以根据schema自动生成对应的类：

	java -jar /path/to/avro-tools-1.8.0.jar compile schema user.avsc .
    
user.avsc的namespace为example.avro，name为User。最终在当前目录生成的example/avro目录下有个User.java文件。

	├── example
    │   └── avro
    │       └── User.java

**使用Avro生成的代码创建User：**

    User user1 = new User();
    user1.setName("Format");
    user1.setFavoriteColor("red");
    user1.setFavoriteNumber(666);

    User user2 = new User("Format2", 66, "blue");

    User user3 = User.newBuilder()
                    .setName("Format3")
                    .setFavoriteNumber(6)
                    .setFavoriteColor("black").build();
    
可以使用有参的构造函数和无参的构造函数，也可以使用Builder构造User。

**序列化：**

DatumWrite接口用来把java对象转换成内存中的序列化格式，SpecificDatumWriter用来生成类并且指定生成的类型。

最后使用DataFileWriter来进行具体的序列化，create方法指定文件和schema信息，append方法用来写数据，最后写完后close文件。

    DatumWriter<User> userDatumWriter = new SpecificDatumWriter<User>(User.class);
            DataFileWriter<User> dataFileWriter = new DataFileWriter<User>(userDatumWriter);
    dataFileWriter.create(user1.getSchema(), new File("users.avro"));
    dataFileWriter.append(user1);
    dataFileWriter.append(user2);
    dataFileWriter.append(user3);
    dataFileWriter.close();

**反序列化：**

反序列化跟序列化很像，相应的Writer换成Reader。这里只创建一个User对象是为了性能优化，每次都重用这个User对象，如果文件量很大，对象分配和垃圾收集处理的代价很昂贵。如果不考虑性能，可以使用 for (User user : dataFileReader) 循环遍历对象

	File file = new File("users.avro");
    DatumReader<User> userDatumReader = new SpecificDatumReader<User>(User.class);
    DataFileReader<User> dataFileReader = new DataFileReader<User>(file, userDatumReader);
    User user = null;
    while(dataFileReader.hasNext()) {
        user = dataFileReader.next(user);
        System.out.println(user);
    }
    
打印出：

    {"name": "Format", "favorite_number": 666, "favorite_color": "red"}
    {"name": "Format2", "favorite_number": 66, "favorite_color": "blue"}
    {"name": "Format3", "favorite_number": 6, "favorite_color": "black"}

## Avro不使用生成的代码进行序列化和反序列化 ##

虽然Avro为我们提供了根据schema自动生成类的方法，我们也可以自己创建类，不使用Avro的自动生成工具。

**创建User：**

首先使用Parser读取schema信息并且创建Schema类：

	Schema schema = new Schema.Parser().parse(new File("user.avsc"));

有了Schema之后可以创建record：

	GenericRecord user1 = new GenericData.Record(schema);
    user1.put("name", "Format");
    user1.put("favorite_number", 666);
    user1.put("favorite_color", "red");

    GenericRecord user2 = new GenericData.Record(schema);
    user2.put("name", "Format2");
    user2.put("favorite_number", 66);
    user2.put("favorite_color", "blue");

使用GenericRecord表示User，GenericRecord会根据schema验证字段是否正确，如果put进了不存在的字段 user1.put("favorite_animal", "cat") ，那么运行的时候会得到AvroRuntimeException异常。

**序列化：**

序列化跟生成的User类似，只不过schema是自己构造的，不是User中拿的。

	Schema schema = new Schema.Parser().parse(new File("user.avsc"));
    GenericRecord user1 = new GenericData.Record(schema);
    user1.put("name", "Format");
    user1.put("favorite_number", 666);
    user1.put("favorite_color", "red");

    GenericRecord user2 = new GenericData.Record(schema);
    user2.put("name", "Format2");
    user2.put("favorite_number", 66);
    user2.put("favorite_color", "blue");

    DatumWriter<GenericRecord> datumWriter = new SpecificDatumWriter<GenericRecord>(schema);
    DataFileWriter<GenericRecord> dataFileWriter = new DataFileWriter<GenericRecord>(datumWriter);
    dataFileWriter.create(schema, new File("users2.avro"));
    dataFileWriter.append(user1);
    dataFileWriter.append(user2);
    dataFileWriter.close();

**反序列化：**

反序列化跟生成的User类似，只不过schema是自己构造的，不是User中拿的。

	Schema schema = new Schema.Parser().parse(new File("user.avsc"));
    File file = new File("users2.avro");
    DatumReader<GenericRecord> datumReader = new SpecificDatumReader<GenericRecord>(schema);
    DataFileReader<GenericRecord> dataFileReader = new DataFileReader<GenericRecord>(file, datumReader);
    GenericRecord user = null;
    while(dataFileReader.hasNext()) {
        user = dataFileReader.next(user);
        System.out.println(user);
    }
    
打印出：

    {"name": "Format", "favorite_number": 666, "favorite_color": "red"}
    {"name": "Format2", "favorite_number": 66, "favorite_color": "blue"}
    
    
## 一些注意点 ##

Avro解析json文件的时候，如果类型是Record并且里面有字段是union并且允许空值的话，需要进行转换。因为["bytes", "string"]和["int","long"]这2个union类型在json中是有歧义的，第一个union在json中都会被转换成string类型，第二个union在json中都会被转换成数字类型。

所以如果json值的null的话，在avro提供的json中直接写null，否则使用只有一个键值对的对象，键是类型，值的具体的值。

比如：

	{
    "namespace": "example.avro",
     "type": "record",
     "name": "User",
     "fields": [
         {"name": "name", "type": "string"},
         {"name": "favorite_number",  "type": ["int","null"]},
         {"name": "favorite_color", "type": ["string","null"]}
     ]
    }

在要转换成json文件的时候要写成这样：

	{"name":"format","favorite_number":{"int":1},"favorite_color":{"string":"red"}}
    {"name":"format2","favorite_number":null,"favorite_color":{"string":"black"}}
	{"name":"format3","favorite_number":{"int":66},"favorite_color":null}

## Spark读取Avro文件 ##

直接遍历avro文件，得到GenericRecord进行处理：

    val conf = new SparkConf().setMaster("local").setAppName("AvroTest")

    val sc = new SparkContext(conf)

    val rdd = sc.hadoopFile[AvroWrapper[GenericRecord], NullWritable, AvroInputFormat[GenericRecord]](this.getClass.getResource("/").toString + "users.avro")

    val nameRdd = rdd.map(s => s._1.datum().get("name").toString)

    nameRdd.collect().foreach(println)
    
    
## 使用Avro需要注意的地方 ##

笔者使用Avro的时候暂时遇到了下面2个坑。先记录一下，以后遇到新的坑会更新这篇文章。

1.如果定义了unions类型的字段，而且unions中有null选项的schema，比如如下schema：

    {
    "namespace": "example.avro",
     "type": "record",
     "name": "User2",
     "fields": [
         {"name": "name", "type": "string"},
         {"name": "favorite_number",  "type": ["null","int"]},
         {"name": "favorite_color", "type": ["null","string"]}
     ]
    }

这样的schema，如果不使用Avro自动生成的model代码进行insert，并且insert中的model数据有null数据的话。然后用spark读avro文件的话，会报org.apache.avro.AvroTypeException: Found null, expecting int ... 这样的错误。

这一点很奇怪，但是使用Avro生成的Model进行insert的话，sprak读取就没有任何问题。 很困惑。


2.如果使用了Map类型的字段，avro生成的model中的Map的Key默认类型为CharSequence。这种model我们insert数据的话，用String是没有问题的。但是spark读取之后要根据Key拿这个Map数据的时候，永远得到的是null。

stackoverflow上有一个页面说到了这个问题。[http://stackoverflow.com/questions/19728853/apache-avro-map-uses-charsequence-as-key
](http://stackoverflow.com/questions/19728853/apache-avro-map-uses-charsequence-as-key
)

需要在map类型的字段里加上"avro.java.string": "String"这个选项, 然后compile的时候使用-string参数即可。

比如以下这个schema：

	{
    "namespace": "example.avro",
     "type": "record",
     "name": "User3",
     "fields": [
         {"name": "name", "type": "string"},
         {"name": "favorite_number",  "type": ["null","int"]},
         {"name": "favorite_color", "type": ["null","string"]},
         {"name": "scores", "type": ["null", {"type": "map", "values": "string", "avro.java.string": "String"}]}
     ]
    }


