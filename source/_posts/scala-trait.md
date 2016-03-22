title: Scala特质(trait)
date: 2015-06-07 20:47:38
tags:
- scala
- jvm
categories: 
- scala
description: Trait就像是有函数体的Interface，跟抽象类类似，trait不像抽象类，它没有构造函数，使用with关键字来混入trait ...

---------------


scala提供了一种叫做trait的特性。

Trait就像是有函数体的Interface，跟抽象类类似，trait不像抽象类，它没有构造函数，使用with关键字来混入trait。

比如想给java.util.ArrayList添加一个foreach方法，这个方法接受一个函数作为参数，对ArrayList重的每个元素执行这个函数。

如果是java的话，只能继承ArrayList，然后添加foreach方法，或者修改ArrayList的源码。

在scala中，可以使用trait完成这个功能。

	trait ForeachAble[A] {
    	def iterator: java.util.Iterator[A]	
        def foreach(f: A => Unit) = {
        	def iter = iterator
            while(iter.hasNext)
            	f(iter.next)
        }
    }
    
    val list = new java.util.ArrayList[Int]() with ForeachAble[Int]
    list.add(1); list.add(2)
    list.foreach({ x => println(x) }) // println 1 and 2
    
    
with关键字可以混入多个trait。

	trait Jsonable {
    	def toJson() = scala.util.parsing.json.JSONFormat.defaultFormatter(this)
    }
    
    val list = new java.util.ArrayList[Int]() with ForeachAble[Int] with Jsonable
    list.add(1); list.add(2)
    list.foreach({ x => println(x) }) // println 1 and 2
    println(list.toJson) // [1, 2]


再来个 + 方法：

	trait Addable[A] {
    	def add(a: A): Boolean
        def +(a: A) = add(a)
    }
    
    val list = new java.util.ArrayList[Int]() with ForeachAble[Int] with Jsonable with Addable[Int]
	list + 2; list + 3
    list.foreach({ x => println(x) }) // println 2 and 3
    println(list.toJson) // [2, 3]

