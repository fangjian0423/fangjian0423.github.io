title: Scala函数式编程
date: 2015-06-12 01:14:55
tags:
- scala
- jvm
categories:
- scala
description: 函数式编程(functional programming)，Lisp作为最古老的函数式编程语言，已重获新生。新的函数式编程语言也层出不穷，比如Erlang，clojure，Scala等... 

---------------

## 函数式编程简介 ##

函数式编程(functional programming)，Lisp作为最古老的函数式编程语言，已重获新生。新的函数式编程语言也层出不穷，比如Erlang，clojure，Scala等。

一个简单的Scala函数式编程例子如下：

	List(1, 2, 3).foreach(x => println(x))

## Scala函数式编程介绍 ##


### 函数是一等公民 ###

函数是第一等公民，指的是函数与其他数据类型一样，处于平等地位，可以赋值给其他变量，也可以作为参数，传入另一个函数，或者作为别的函数的返回值。

以下就是将一个匿名函数赋值给++变量的例子：

	val ++ = (num: Int) => num + 1
    
	++(2) // 3
    ++(6) // 7
    
    // ++ 变量赋值给tmp变量
    val tmp = ++
    tmp(7) // 8
    tmp(9) // 10
    
函数作为参数例子，一个最典型的例子就是一些connection的处理，connection处理完毕之后都需要close，可以使用函数作为参数，这样就不用每次都close数据了：

	import scala.reflect.io.File
    import java.util.Scanner
    
    def scanFile(f: File, f: Scanner => Unit) = {
    	val scanner = new Scanner(f.bufferedReader)
        try {
        	f(scanner)
        } finally {
        	scanner.close()
        }
    }
    
    scanFile(File("/tmp/test.txt"), scanner => println(scanner.next()))

### 一些常用的higher-order函数 ###

higher-order函数指的是以函数做参数或者返回值是一个函数的函数。

List中的map，flatMap，foreach函数都是higher-order函数。

map例子，map函数的作用是使用一个函数调用集合中的各个元素，得到一个新的集合：

	List(1, 2, 3).map({x => x * 2})  // List(2, 4, 6)
    
可以简化为：

	List(1, 2, 3).map(x => x * 2)  // List(2, 4, 6)
    
还可以简化为:

	List(1, 2, 3).map { x => x * 2 } // List(2, 4, 6)  currying函数的应用
    
还可以简化为：

	List(1, 2, 3).map { _ * 2 } // List(2, 4, 6)
    
flatMap例子，flatMap函数的作用跟map函数类似，把多个集合合并成一个集合，注意，**flatMap针对的是集合**：

	List(1, 2, 3).flatMap { x => List(x * 2) } // List(2, 4, 6)
    
foreach例子：

	List(1, 2, 3).foreach { println(_) }

	
### 函数和方法的区别 ###

方法：方法指的是定义在类中的方法

	class UseResource {
    	def use(r: Resource): Boolean = { ... }
    }
    
上面UseResource类中的use方法就是一个方法，it is a method.

函数：函数在scala中代表1个类型和一个对象，方法却不会，方法只会出现在类中。

	val succ = (x: Int) => x + 1
    
scala允许将方法转换成函数，可以在方法后面添加 "_" 即可。比如：

	val use_func: Resource => Boolean = new UserResource().use _

### 函数柯里化(function currying) ###

函数柯里化是一种把函数中多个参数改造成只有一个参数的技术。

	def add(x: Int, y: Int) = x + y
    
add函数柯里化之后：

	def add(x: Int) = (y: Int) => x + y
    
简化为:

	def add(x: Int)(y: Int) = x + y
    
调用柯里化后的add函数：

	add(2)(3) // 5
    add(2) { 5 } // 7


scala支持把一个非柯里化的函数转换成一个柯里化函数，使用函数变量的curried方法：

	def add(x: Int, y: Int) = x + y
    val addVar = add _ // 转换成函数变量
    val curryingFunc = addVar.curried // 使用curried方法转换成柯里化函数
    curryingFunc(1) { 3 } // 4
    
	// 或者定义add函数的时候直接使用函数变量
    val add = (x: Int, y: Int) => x + y
    val curryingFunc = addVar.curried
    curryingFunc(1) { 3 } // 4


不过一般我们定义函数的时候都会定义成柯里化函数，而不会去转换：

	def add(x: Int)(y: Int) = x + y
    add(2) { 7 } // 9
    
	// 实现一个map函数
    def map[A, B](xs: List[A])(func: A => B) = xs.map { func(_) }
    // List[String] = List(11, 21, 31)
    map(List(1, 2, 3)) {
    	x => x + "1"
    }
    
## 鸭子类型 ##

鸭子类型是动态类型的一种风格，使用鸭子类型就不可以不使用继承这种不够灵活的特性。

	def withClose(closeAble: { def close(): Unit }, op: { def close(): Unit } => Unit) {
    	try {
        	op(closeAble)
        } finally {
        	closeAble.close()
        }
    }
    
    class Connection {
    	def close = println("close Connection")
    }
    val conn = new Connection()
    withClose(conn, conn => println("do something with Connection"))
    





