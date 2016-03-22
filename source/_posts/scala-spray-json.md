title: spray-json源码分析
date: 2015-12-23 01:47:35
tags:
- scala
- jvm
categories:
- scala
description: spray-json是scala的一个轻量的，简洁的，简单的关于JSON实现。同时也是spray项目的json模块，本文分析spray-json的源码 ...

--------------

## spray-json介绍以及内部结构 ##

[spray-json](https://github.com/spray/spray-json)是scala的一个轻量的，简洁的，简单的关于JSON实现。

同时也是[spray](http://spray.io/)项目的json模块。

本文分析spray-json的源码。

在分析spray-json的源码之前，我们先介绍一下spray-json的使用方法以及里面的几个概念。

首先是spray-json的一个使用例子，里面有各种黑魔法：

	val str = """{
        "name": "Ed",
	    "age": 24
    }"""

	// 黑魔法。不是String的parseJson方法，而是使用了隐式转换，隐式转换成PimpedString类。PimpedString里有parseJson方法，转换成JsValue对象
    val jsonVal = str.parseJson // jsonVal是个JsObject对象的实例

	// jsonVal是个JsObject对象，也是个JsValue实例。JsValue对象都有compactPrint和prettyPrint方法
    println(jsonVal.compactPrint) // 压缩打印
    println(jsonVal.prettyPrint) // 格式化打印
    
    // 手动构建一个JsObject
    val jsonObj = JsObject(
    	("name", JsString("format")), ("age", JsNumber(99))
    )

    println(jsonObj.compactPrint)
    println(jsonObj.prettyPrint)
    
    // 黑方法，不是List的toJson方法，而是使用了隐式转换，隐式转换成PimpedAny类，PimpedAny类里有toList方法，转换成对应的类型
    val jsonList = List(1,2,3).toJson
    
	// JsValue的toString方法引用了compactPrint方法
	println(jsonList)


然后是spray-json里的几个概念介绍：

1.JsValue: 抽象类。对原生json中的各种数据类型的抽象。它的实现类JsArray对应json里的数组；JsBoolean对应json里的布尔值；JsNumber对应json里的数值 ...

2.JsonFormat: 一个trait，是json序列化和反序列话的抽象。继承JsonReader(JsValue转换成对象的抽象)和JsonWriter(对象转换成JsValue的抽象)。BasicFormats和CollectionFormats、AdditionalFormats等这些trait里都有各种JsonFormat的隐式转换。

3.RootJsonFormat。json文档的抽象，跟JsonFormat一样，只不过RootJsonFormat只支持JsArray和JsObject。因为这是一个json文档对象，只有一个json对象或者一个json数组才能称得上是一个json文档对象。

4.JsonPrinter: 一个trait。json打印成字符串的抽象。具体的实现特质有CompactPrinter(压缩后的字符串)和PrettyPrinter(格式化后的字符串)

5.DefaultJsonProtocol: 继承BasicFormats，混入StandardFormats、CollectionFormats、ProductFormats、AdditionalFormats的特质。我们需要转换一些基础类型或者集合类型的时候需要import这个trait。

## json package对象 ##

json package对象里定义了一些隐式转换方法和一些实用方法。

    package object json {

	  // JsField。 一个二元元组，代表json中的一个项(key为String，value为任意json类型)
      type JsField = (String, JsValue)

	  // 反序列化异常
      def deserializationError(msg: String, cause: Throwable = null, fieldNames: List[String] = Nil) = throw new DeserializationException(msg, cause, fieldNames)
      // 序列化异常
      def serializationError(msg: String) = throw new SerializationException(msg)
	  // jsonReader方法，是个泛型。使用了隐式参数，返回值是这个隐式参数的引用。也就是说只要调用了jsonReader方法，那么就会自动去找对应泛型类型的实现
      def jsonReader[T](implicit reader: JsonReader[T]) = reader
      // 跟jsonReader方法一个道理。只要调用了jsonWriter方法，那么就会自动去找对应泛型类型的实现
      def jsonWriter[T](implicit writer: JsonWriter[T]) = writer 

	  // 隐式转换方法。上面例子的toList使用了这个隐式转换
      implicit def pimpAny[T](any: T) = new PimpedAny(any)
      // 隐式方法。上面例子的parseJson使用了这个隐式转换
      implicit def pimpString(string: String) = new PimpedString(string)
    }
    
    package json {
	
      // 反序列异常类的定义，上面的deserializationError方法实例化了这个类
      case class DeserializationException(msg: String, cause: Throwable = null, fieldNames: List[String] = Nil) extends RuntimeException(msg, cause)
      // 序列异常类的定义，上面的serializationError方法实例化了这个类
      class SerializationException(msg: String) extends RuntimeException(msg)

	  // 上面的隐式方法pimpAny实例化了这个类。黑魔法toJson方法，不是List的toJson方法，而是List隐式转换成PimpedAny，然后调用PimpedAny的toJson方法。toJson方法的参数是个隐式参数，跟上面代码里的jsonWriter方法一样，会找对应泛型类型的JsonWriter实现类，然后调用JsonWriter的write方法
      private[json] class PimpedAny[T](any: T) {
        def toJson(implicit writer: JsonWriter[T]): JsValue = writer.write(any)
      }

	  // 上面的隐式方法pimpString实例化了这个类。黑魔法parseJson方法，不是String的parseJson方法，而是String隐式转换成PimpedString，然后调用PimpedString的parseJson方法
      private[json] class PimpedString(string: String) {
        @deprecated("deprecated in favor of parseJson", "1.2.6")
        def asJson: JsValue = parseJson
        def parseJson: JsValue = JsonParser(string)
      }
    }

## JsValue(原生json中的各种数据类型的抽象) ##

JsValue是原生json中各种数据类型的抽象，是个抽象类，直接看JsValue的定义:

    sealed abstract class JsValue {
   	  // 重载的toString方法引用了compactPrint方法，会打印出json的压缩格式
      override def toString = compactPrint
      def toString(printer: (JsValue => String)) = printer(this)
      // 压缩打印，使用了CompactPrinter。CompactPrinter是一个JsonPrinter的子类
      def compactPrint = CompactPrinter(this)
      // 格式化打印，PrettyPrinter。PrettyPrinter也是一个JsonPrinter的子类
      def prettyPrint = PrettyPrinter(this)
      // 转换成对应的类。jsonReader方法在json package里定义，已经分析过。会找对应那个的JsonReader实现类。然后调用read方法
      def convertTo[T :JsonReader]: T = jsonReader[T].read(this)

	  // 转换成JsObject对象，除了JsObject对象重写了这个，返回了自身。其他类型的JsValue都会抛出DeserializationException异常
      def asJsObject(errorMsg: String = "JSON object expected"): JsObject = deserializationError(errorMsg)

      def asJsObject: JsObject = asJsObject()

      @deprecated("Superceded by 'convertTo'", "1.1.0")
      def fromJson[T :JsonReader]: T = convertTo
    }

JsNumber是数值类型的抽象：

	case class JsNumber(value: BigDecimal) extends JsValue
    
    // JsNumber中定义了几个方便的构造方法
    object JsNumber {
	  val zero: JsNumber = apply(0)
	  def apply(n: Int) = new JsNumber(BigDecimal(n))
	  def apply(n: Long) = new JsNumber(BigDecimal(n))
	  def apply(n: Double) = n match {
	    case n if n.isNaN      => JsNull
	    case n if n.isInfinity => JsNull
    	case _                 => new JsNumber(BigDecimal(n))
      }
      def apply(n: BigInt) = new JsNumber(BigDecimal(n))
      def apply(n: String) = new JsNumber(BigDecimal(n))
      def apply(n: Array[Char]) = new JsNumber(BigDecimal(n))
	}

JsString是字符串类型的抽象：

	case class JsString(value: String) extends JsValue
    
    // JsString中定义了几个方便的方法
    object JsString {
        val empty = JsString("")
        def apply(value: Symbol) = new JsString(value.name)
    }

JsObject是对象类型的抽象：

	case class JsObject(fields: Map[String, JsValue]) extends JsValue {
      // 重写了asJsObject方法
      override def asJsObject(errorMsg: String) = this
      // 根据字段名获得对应的JsValue
      def getFields(fieldNames: String*): immutable.Seq[JsValue] = fieldNames.flatMap(fields.get)(collection.breakOut)
	}
    
    object JsObject {
      val empty = JsObject(Map.empty[String, JsValue])
      // 使用多个JsField构造JsObject。这里的JsField就是代表一个(String, JsValue)
      def apply(members: JsField*) = new JsObject(Map(members: _*))
      @deprecated("Use JsObject(JsValue*) instead", "1.3.0")
      def apply(members: List[JsField]) = new JsObject(Map(members: _*))
    }
    
## JsonFormat(JsonWriter和JsonReader的子类) ##

一个trait，是json序列化和反序列话的抽象。继承JsonReader(JsValue转换成对象的抽象)和JsonWriter(对象转换成JsValue的抽象)。

JsonReader的定义，把一个JsValue转换成对应的类型：

    trait JsonReader[T] {
      def read(json: JsValue): T
    }

JsonWriter的定义，把一个类型转换成JsValue：

    trait JsonWriter[T] {
      def write(obj: T): JsValue
    }

json package对象里的jsonReader和jsonWriter方法有个隐式参数，我们也分析过：只要调用了jsonReader(JsonWriter)方法，那么就会自动去找对应泛型类型的实现。

AdditionalFormats、BasicFormats、CollectionFormats、StandardFormats等都定义了各种JsonFormat。

比如Int类型就找IntJsonFormat，String类型就找StringJsonFormat ...  这些基础类型的JsonFormat都定义在BasicFormats这个trait中。

我们就分析几个BasicFormats中定义的基础类型JsonFormat：

Int基本类型的JsonFormat：

	implicit object IntJsonFormat extends JsonFormat[Int] {
    	// write方法继承自JsonWriter。 直接实例化一个JsNumber对象
    	def write(x: Int) = JsNumber(x)
        // read方法继承自JsonReader。读取JsNumber中对应的值
	    def read(value: JsValue) = value match {
    	  case JsNumber(x) => x.intValue
	      case x => deserializationError("Expected Int as JsNumber, but got " + x)
	    }
    }
    
String基本类型的JsonFormat：
    
    implicit object StringJsonFormat extends JsonFormat[String] {
    	// write方法实例化一个JsString
        def write(x: String) = {
          require(x ne null)
          JsString(x)
        }
        // read方法读取JsString中对应的字符串
        def read(value: JsValue) = value match {
          case JsString(x) => x
          case x => deserializationError("Expected String as JsString, but got " + x)
        }
	}
    
.....


CollectionFormats中定义了几个集合类的JsonFormat:

List类型的JsonFormat：

	implicit def listFormat[T :JsonFormat] = new RootJsonFormat[List[T]] {
    	// 将List转换成JsArray对象。遍历list中的各个元素，对每个元素调用toJson方法。最后JsArray里的每个元素都是JsValue
        def write(list: List[T]) = JsArray(list.map(_.toJson).toVector)
        // JsArray转换成List。对JsArray里的各个JsValue调用convertTo转换成对应的类型
        def read(value: JsValue): List[T] = value match {
          case JsArray(elements) => elements.map(_.convertTo[T])(collection.breakOut)
          case x => deserializationError("Expected List as JsArray, but got " + x)
        }
    }

Map类型的JsonFormat：

	implicit def mapFormat[K :JsonFormat, V :JsonFormat] = new RootJsonFormat[Map[K, V]] {
    	// 遍历Map中的每一个二元元组。如果元组的第一项不是String，直接抛出SerializationException异常。否则构造key为元组第一项字符串，value为元组第二项的JsVaue对象。
        def write(m: Map[K, V]) = JsObject {
          m.map { field =>
            field._1.toJson match {
              case JsString(x) => x -> field._2.toJson
              case x => throw new SerializationException("Map key must be formatted as JsString, not '" + x + "'")
            }
          }
        }
        def read(value: JsValue) = value match {
          case x: JsObject => x.fields.map { field =>
            (JsString(field._1).convertTo[K], field._2.convertTo[V])
          } (collection.breakOut)
          case x => deserializationError("Expected Map as JsObject, but got " + x)
        }
    }

AdditionalFormats中定义了一些helper和额外的JsonFormat：

	// JsValue的JsonFormat，JsValue调用convertTo或者toJson返回的就是自身
	implicit object JsValueFormat extends JsonFormat[JsValue] {
      def write(value: JsValue) = value
      def read(value: JsValue) = value
    }

	
    implicit object RootJsObjectFormat extends RootJsonFormat[JsObject] {
      def write(value: JsObject) = value
      def read(value: JsValue) = value.asJsObject
    }

StandardFormats中定义了Option、Either的JsonFormat，1元-7元元组的JsonFormat。

    implicit def tuple1Format[A :JF] = new JF[Tuple1[A]] {
      def write(t: Tuple1[A]) = t._1.toJson
      def read(value: JsValue) = Tuple1(value.convertTo[A])
    }

    implicit def tuple2Format[A :JF, B :JF] = new RootJsonFormat[(A, B)] {
      def write(t: (A, B)) = JsArray(t._1.toJson, t._2.toJson)
      def read(value: JsValue) = value match {
        case JsArray(Seq(a, b)) => (a.convertTo[A], b.convertTo[B])
        case x => deserializationError("Expected Tuple2 as JsArray, but got " + x)
      }
    }


## JsonPrinter(将JsValue打印成原生json字符串) ##

JsonPrinter继承一个函数对象，这个函数的输入是个JsValue，输出是String：

	trait JsonPrinter extends (JsValue => String)
    
内部定义了一个抽象方法：

	def print(x: JsValue, sb: JStringBuilder)
    

CompactPrinter继承JsonPrinter，压缩打印：

实现的print方法：

	def print(x: JsValue, sb: StringBuilder) {
        x match {
          case JsObject(x) => printObject(x, sb)
          case JsArray(x)  => printArray(x, sb)
          case _ => printLeaf(x, sb)
        }
    }
    
PrettyPrinter也继承JsonPrinter，格式化打印：

实现的print方法：

	def print(x: JsValue, sb: StringBuilder) {
      print(x, sb, 0)
    }
  
  	// indent参数是格式化打印的关键
    protected def print(x: JsValue, sb: StringBuilder, indent: Int) {
      x match {
        case JsObject(x) => printObject(x, sb, indent)
        case JsArray(x)  => printArray(x, sb, indent)
        case _ => printLeaf(x, sb)
      }
    }
    
## DefaultJsonProtocol(整合了多个JsonFormat) ##

直接看源码：

	trait DefaultJsonProtocol
        extends BasicFormats
        with StandardFormats
        with CollectionFormats
        with ProductFormats
        with AdditionalFormats
        
	object DefaultJsonProtocol extends DefaultJsonProtocol
