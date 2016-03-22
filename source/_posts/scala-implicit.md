title: Scala 隐式转换和隐式参数
date: 2015-12-20 15:38:22
tags:
- scala
- jvm
categories:
- scala
description: Scala的implicit功能很强大，可以自动地给对象"添加一个属性"。 这里打上引号的原因是Scala内部进行编译的时候会自动加上隐式转换函数 ...

--------------

## 前言 ##

Scala的implicit功能很强大，可以自动地给对象"添加一个属性"。 这里打上引号的原因是Scala内部进行编译的时候会自动加上隐式转换函数。

很多Scala开源框架内部都大量使用了implicit。因为implicit真的很强大，写得好的implicit可以让代码更优雅。但个人感觉implicit也有一些缺点，比如使用了implicit之后，看源码或者使用一些library的时候无法下手，因为你根本不知道作者哪里写了implicit。这个也会对初学者造成一些困扰。

比如Scala中Option就有一个implicit可以将Option转换成Iterable：

	val list = List(1, 2)
    val map = Map(1 -> 11, 2 -> 22, 3 -> 33)
    
    val newList = list.flatMap {
    	num => map.get(num) // map.get方法返回的是Option，可以被隐式转换成Iterable
    } 

以下是implicit的一个小例子。

比如以下一个例子，定义一个Int类型的变量num，但是赋值给了一个Double类型的数值。这时候就会编译错误：

	val num: Int = 3.5 // Compile Error
    
但是我们加了一个隐式转换之后，就没问题了:

	implicit def double2Int(d: Double) = d.toInt
    
    val num: Int = 3.5 // 3， 这段代码会被编译成 val num: Int = double2Int(3.5)
    

## 隐式转换规则 ##

### 标记规则(Marking Rule) ###

任何变量，函数或者对象都可以用implicit这个关键字进行标记，表示可以进行隐式转换。

	implicit def intToString(x: Int) = x.toString

编译器可能会将x + y 转换成 convert(x) + y 如果convert被标记成implicit。
    
### 作用域规则(Scope Rule) ###

在一个作用域内，一个隐式转换必须是一个唯一的标识。

比如说MyUtils这个object里有很多隐式转换。x + y 不会使用MyUtils里的隐式转换。 除非import进来。 import MyUtils._

Scala编译器还能在companion class中去找companion object中定义的隐式转换。

	object Player {
  	  implicit def getClub(player: Player): Club = Club(player.clubName)
    }

    class Player(val name: String, val age: Int, val clubName: String) {

    }

    val p = new Player("costa", 27, "Chelsea")

    println(p.welcome) // Chelsea welcome you here!
    println(p.playerNum) // 21


### 一次编译只隐式转换一次(One-at-a-time Rule) ###

Scala不会把 x + y 转换成 convert1(convert2(x)) + y


## 隐式转换类型 ##

### 隐式转换成正确的类型 ###

这种类型是Scala编译器对隐式转换的第一选择。 比如说编译器看到一个类型的X的数据，但是需要一个类型为Y的数据，那么就会去找把X类型转换成Y类型的隐式转换。

本文一开始的double2Int方法就是这种类型的隐式转换。

	implicit def double2Int(d: Double) = d.toInt
    
    val num: Int = 3.5 // 3
    
当编译器发现变量num是个Int类型，并且用Double类型给它赋值的时候，会报错。 但是在报错之前，编译器会查找Double => Int的隐式转换。然后发现了double2Int这个隐式转换函数。于是就使用了隐式转换。

### 方法调用的隐式转换 ###

比如这段代码  obj.doSomeThing。 比如obj对象没有doSomeThing这个方法，编译器会会去查找拥有doSomeThing方法的类型，并且看obj类型是否有隐式转换成有doSomeThing类型的函数。有的话就是将obj对象隐式转换成拥有doSomeThing方法的对象。

以下是一个例子：

	case class Person(name: String, age: Int) {
	    def +(num: Int) = age + num
    	def +(p: Person) = age + p.age
  	}
    
    val person = Person("format", 99)
    println(person + 1) // 100
	//  println(1 + person)  报错，因为Int的+方法没有有Person参数的重载方法
    
    implicit def personAddAge(x: Int) = Person("unknown", x)
    
    println(1 + person) // 100
    
有了隐式转换方法之后，编译器检查 1 + person 表达式，发现Int的+方法没有有Person参数的重载方法。在放弃之前查看是否有将Int类型的对象转换成以Person为参数的+方法的隐式转换函数，于是找到了，然后就进行了隐式转换。

Scala的Predef中也使用了方法调用的隐式转换。

	Map(1 -> 11, 2 -> 22)
    
上面这段Map中的参数是个二元元组。 Int没有 -> 方法。 但是在Predef中定义了：

	implicit final class ArrowAssoc[A](private val self: A) extends AnyVal {
    	@inline def -> [B](y: B): Tuple2[A, B] = Tuple2(self, y)
	    def →[B](y: B): Tuple2[A, B] = ->(y)
    }
    
### 隐式参数 ###

隐式参数的意义是当方法需要多个参数的时候，可以定义一些隐式参数，这些隐式参数可以被自动加到方法填充的参数里，而不必手填充。

	def implicitParamFunc(name: String)(implicit tiger: Tiger, lion: Lion): Unit = {
  	  println(name + " have a tiget and a lion, their names are: " + tiger.name + ", " + lion.name)
    }

	object Zoo {
    	implicit val tiger = Tiger("tiger1")
    	implicit val lion = Lion("lion1")
    }
    
    import Zoo._

	implicitParamFunc("format")

上面这个代码中implicitParamFunc中的第二个参数定义成了隐式参数。

然后在Zoo对象里定义了两个隐式变量，import进来之后，调用implicitParamFunc方法的时候这两个变量被自动填充到了参数里。

这里需要注意的是不仅仅方法中的参数需要被定义成隐式参数，对应的隐式参数的变量也需要被定义成隐式变量。


## 其他 ##

对象中的隐式转换可以只import自己需要的。

	object MyUtils {
		implicit def a ...
        implicit def b ...
	}
    
    import MyUtils.a
    
隐式转换修饰符implicit可以修饰class，method，变量，object。

修饰方法和变量的隐式转换本文已经介绍过，就不继续说了。


修饰class的隐式转换，它的作用跟修饰method的隐式转换类似：

	implicit class RangeMarker(val start: Int) {
  	  def -->(end: Int) = start to end
    }
    
    1 --> 10 // Range(1, 10)
    
上段代码可以改造成使用Value Class完成类的隐式转换：

	implicit class RangeMaker(start: Int) extends AnyVal {
    	def -->(end: Int) = start to end
	}

修饰object的隐式转换：

	trait Calculate[T] {
	    def add(x: T, y: T): T
    }

    implicit object IntCal extends Calculate[Int] {
    	def add(x: Int, y: Int): Int = x + y
    }

    implicit object ListCal extends Calculate[List[Int]] {
    	def add(x: List[Int], y: List[Int]): List[Int] = x ::: y
    }

    def implicitObjMethod[T](x: T, y: T)(implicit cal: Calculate[T]): Unit = {
	    println(x + " + " + y + " = " + cal.add(x, y))
    }

    implicitObjMethod(1, 2) // 1 + 2 = 3
    implicitObjMethod(List(1, 2), List(3, 4)) // List(1, 2) + List(3, 4) = List(1, 2, 3, 4)


