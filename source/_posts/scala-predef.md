title: Scala的Predef介绍
date: 2015-10-07 00:37:20
tags:
- scala
- jvm
categories: 
- scala
description: Scala每个程序都会自动import以下3个包，分别是 java.lang._  ,  scala._ 和 Predef._ 。跟Java程序都会自动import java.lang一样 ...

---------------

## 前言 ##

Scala每个程序都会自动import以下3个包，分别是 java.lang._  ,  scala._ 和 Predef._ 。跟Java程序都会自动import java.lang一样。

Predef是一个单例对象，所以我们import进来之后，可以直接使用Predef中定义的方法。

## Predef中定义的方法和属性 ##


### 常用方法和类 ###

	def classOf[T]: Class[T] = null // This is a stub method. The actual implementation is filled in by the compiler.

    type String        = java.lang.String  // scala中的String使用jdk中的String
    type Class[T]      = java.lang.Class[T] // scala中的Class使用jdk中的Class
    
    type Function[-A, +B] = Function1[A, B] // Function1取别名Function

    type Map[A, +B] = immutable.Map[A, B] // Map类型默认使用immutable包下的Map
    type Set[A]     = immutable.Set[A] // Set类型默认使用immutable包下的Set
    val Map         = immutable.Map // Map对象默认使用immutable包下的Map对象(下面测试用到)
    val Set         = immutable.Set // Set对象默认使用immutable包下的Set对象(下面测试用到)
    
一些测试：

	// 默认的Map和Set都是immutable包下的，这里的Set和Map都是在Predef中定义的一个变量。
	Map(1 -> 1, 2 -> 2) // scala.collection.immutable.Map[Int,Int] = Map(1 -> 1, 2 -> 2)
    Set(1,2,3) // scala.collection.immutable.Set[Int] = Set(1, 2, 3)
    
    
### 打印方法 ###

	def print(x: Any) = Console.print(x)
    def println() = Console.println()
    def println(x: Any) = Console.println(x)
    def printf(text: String, xs: Any*) = Console.print(text.format(xs: _*))
    
因此，平时我们使用的println，print，printf这些打印方法都是在Predef中定义的，

### 一些调试和错误方法 ###
  
	// 过期方法，抛出带有message消息的RuntimeException
    @deprecated("Use `sys.error(message)` instead", "2.9.0")
    def error(message: String): Nothing = sys.error(message)

	// 断言。 参数是一个Boolean类型，失败抛出java.lang.AssertionError异常
    @elidable(ASSERTION)
    def assert(assertion: Boolean) {
    if (!assertion)
      throw new java.lang.AssertionError("assertion failed")
    }

	// 跟上一个方法类似，多了一个message参数。抛出的异常就打印出这个message参数
    @elidable(ASSERTION) @inline
    final def assert(assertion: Boolean, message: => Any) {
    if (!assertion)
      throw new java.lang.AssertionError("assertion failed: "+ message)
    }

	// 跟assert类似，唯一的区别是assume支持静态经验(static checker)
    @elidable(ASSERTION)
    def assume(assumption: Boolean) {
    if (!assumption)
      throw new java.lang.AssertionError("assumption failed")
    }

	// 跟assume一样，多了个message参数
    @elidable(ASSERTION) @inline
    final def assume(assumption: Boolean, message: => Any) {
    if (!assumption)
      throw new java.lang.AssertionError("assumption failed: "+ message)
    }

	// 跟assert类似，只不过抛出的是IllegalArgumentException异常
    def require(requirement: Boolean) {
    if (!requirement)
      throw new IllegalArgumentException("requirement failed")
    }

	// 多个参数，作用如上一样
    @inline final def require(requirement: Boolean, message: => Any) {
    if (!requirement)
      throw new IllegalArgumentException("requirement failed: "+ message)
    }

调试和错误方法测试：

	assert(1 == 2) // java.lang.AssertionError: assertion failed
    assert(1 == 2, "test") // java.lang.AssertionError: assertion failed: test
    assume(1 == 2) // java.lang.AssertionError: assumption failed
    assume(1 == 2, "test") // java.lang.AssertionError: assumption failed: test
    require(1 == 2) // java.lang.IllegalArgumentException: requirement failed
    require(1 == 2, "test") // java.lang.IllegalArgumentException: requirement failed: test


### 一个特殊的属性 ###

Predef中有个 ??? 属性，抛出一个NotImplementedError：

	def ??? : Nothing = throw new NotImplementedError
    
比如定义一些方法的时候，这个方法还没有实现，这个时候可以使用 ???， 而非TODO：

	def todoMethod(x: Int): Int = ???
    
    todoMethod(2) // scala.NotImplementedError: an implementation is missing
    

## Predef还有大量的隐式转换和隐式转换类 ##

再讲Predef中的隐式转换和隐式转换类之前，先介绍一下这2个概念。
    
### 隐式转换 ###

隐式转换的意思是一个方法中有一个类型的参数，并返回另外一个类型的返回值。比如一个Double类型的方法返回一个Int类型的返回值。

	def double2Int(d: Double) = d.toInt
    
    double2Int(2.4) // 2
    
    val a: Int = 2.3 // 报错
    
重新定义double2Int，使其支持隐式转换：

	implicit def double2Int(d: Double) = d.toInt
    
    val a: Int = 2.3 // a: Int = 2
    
### 隐式转换类 ###

当需要给Int类型添加一个 --> 方法的时候，需要使用到隐式转换类。因为隐式转换只支持1个参数，所以只能通过隐式转换类完成。

	implicit class RangeMaker(left: Int) {
    	def -->(right: Int) = left to right
    }
    
    val range = 1 --> 10 // Range(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
    
隐式转换类有个弊端，那就是每次都会创建一个类的实例，有时候这完全没有必要。

scala提供了一个将隐式转换类转换成value class的方法，只需要继承AnyVal即可，还需要注意参数需要加上val标识符。

	implicit class RangeMaker(val left: Int) extends AnyVal {
    	def -->(right: Int) = left to right
    }

### Predef中的隐式转换和隐式转换类 ###

-> 方法：

	implicit final class ArrowAssoc[A](private val self: A) extends AnyVal {
        @inline def -> [B](y: B): Tuple2[A, B] = Tuple2(self, y)
	    def →[B](y: B): Tuple2[A, B] = ->(y)
    }
    
生成一个二元元组，Tuple2。

	1 → 2  // (Int, Int) = (1, 2)
    1 -> 2 // (Int, Int) = (1, 2)
    

ensuring 方法：

	implicit final class Ensuring[A](private val self: A) extends AnyVal {
        def ensuring(cond: Boolean): A = { assert(cond); self }
        def ensuring(cond: Boolean, msg: => Any): A = { assert(cond, msg); self }
        def ensuring(cond: A => Boolean): A = { assert(cond(self)); self }
        def ensuring(cond: A => Boolean, msg: => Any): A = { assert(cond(self), msg); self }
	}

ensuring内部调用assert方法。

	def doublePositive(n: Int): Int = {
    	n * 2
    } ensuring(n => n >= 0 && n % 2 == 0)
    
    doublelPositive(1) // 2
    doublelPositive(-1) // java.lang.AssertionError: assertion failed
    
    
formatted 方法：

	implicit final class StringFormat[A](private val self: A) extends AnyVal {
  	  @inline def formatted(fmtstr: String): String = fmtstr format self
    }
    
    
格式化字符串，使用java.lang.String.format方法。

	"Format" formatted "%s, let's go" // Format, let's go
	

\+ 方法：

	implicit final class any2stringadd[A](private val self: A) extends AnyVal {
  	  def +(other: String): String = String.valueOf(self) + other
    }
    
case class使用\+方法：

	case class Student(name: String, age: Int)
    Student("format", 11) + " 22" // Student(format,11) 22
    
    
StringOps类：
    
	@inline implicit def augmentString(x: String): StringOps = new StringOps(x)
    @inline implicit def unaugmentString(x: StringOps): String = x.repr

StringOps提供了丰富的原生String没提供的方法：

	"format".length // 6 原生String是没有提供length方法的
    "format".foreach(println(_))
    "format".stripPrefix("for") // mat
    "format".slice(1, 3) // or
    "format" * 5 // formatformatformatformatformat
    "true".toBoolean // true
    

ArrayOps类：

	implicit def genericArrayOps[T](xs: Array[T]): ArrayOps[T] = (xs match {
        case x: Array[AnyRef]  => refArrayOps[AnyRef](x)
        case x: Array[Boolean] => booleanArrayOps(x)
        case x: Array[Byte]    => byteArrayOps(x)
        case x: Array[Char]    => charArrayOps(x)
        case x: Array[Double]  => doubleArrayOps(x)
        case x: Array[Float]   => floatArrayOps(x)
        case x: Array[Int]     => intArrayOps(x)
        case x: Array[Long]    => longArrayOps(x)
        case x: Array[Short]   => shortArrayOps(x)
        case x: Array[Unit]    => unitArrayOps(x)
        case null              => null
    }).asInstanceOf[ArrayOps[T]]
    
    
      implicit def booleanArrayOps(xs: Array[Boolean]): ArrayOps[Boolean] = new ArrayOps.ofBoolean(xs)
      implicit def byteArrayOps(xs: Array[Byte]): ArrayOps[Byte]          = new ArrayOps.ofByte(xs)
      implicit def charArrayOps(xs: Array[Char]): ArrayOps[Char]          = new ArrayOps.ofChar(xs)
      implicit def doubleArrayOps(xs: Array[Double]): ArrayOps[Double]    = new ArrayOps.ofDouble(xs)
      implicit def floatArrayOps(xs: Array[Float]): ArrayOps[Float]       = new ArrayOps.ofFloat(xs)
      implicit def intArrayOps(xs: Array[Int]): ArrayOps[Int]             = new ArrayOps.ofInt(xs)
      implicit def longArrayOps(xs: Array[Long]): ArrayOps[Long]          = new ArrayOps.ofLong(xs)
      implicit def refArrayOps[T <: AnyRef](xs: Array[T]): ArrayOps[T]    = new ArrayOps.ofRef[T](xs)
      implicit def shortArrayOps(xs: Array[Short]): ArrayOps[Short]       = new ArrayOps.ofShort(xs)
      implicit def unitArrayOps(xs: Array[Unit]): ArrayOps[Unit]          = new ArrayOps.ofUnit(xs)

ArrayOps提供了丰富的原生数组没提供的方法：

	Array(1, 2, 3) :+ 4 // Array(1, 2, 3, 4)
    
    
Scala程序员可以较少关心装箱和拆箱操作，这也是由于Predef对象里定义了Scala值类型与java基本类型直接的隐式转换：

	implicit def byte2Byte(x: Byte)           = java.lang.Byte.valueOf(x)
    implicit def short2Short(x: Short)        = java.lang.Short.valueOf(x)
    implicit def char2Character(x: Char)      = java.lang.Character.valueOf(x)
    implicit def int2Integer(x: Int)          = java.lang.Integer.valueOf(x)
    implicit def long2Long(x: Long)           = java.lang.Long.valueOf(x)
    implicit def float2Float(x: Float)        = java.lang.Float.valueOf(x)
    implicit def double2Double(x: Double)     = java.lang.Double.valueOf(x)
    implicit def boolean2Boolean(x: Boolean)  = java.lang.Boolean.valueOf(x)

    implicit def Byte2byte(x: java.lang.Byte): Byte             = x.byteValue
    implicit def Short2short(x: java.lang.Short): Short         = x.shortValue
    implicit def Character2char(x: java.lang.Character): Char   = x.charValue
    implicit def Integer2int(x: java.lang.Integer): Int         = x.intValue
    implicit def Long2long(x: java.lang.Long): Long             = x.longValue
    implicit def Float2float(x: java.lang.Float): Float         = x.floatValue
    implicit def Double2double(x: java.lang.Double): Double     = x.doubleValue
    implicit def Boolean2boolean(x: java.lang.Boolean): Boolean = x.booleanValue
    
