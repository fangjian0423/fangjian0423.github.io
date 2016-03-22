title: Scala偏函数(Partial Function)和部分应用(Partial application)
date: 2015-06-14 22:37:38
tags:
- scala
- jvm
categories:
- scala
description: Scala的偏函数支持2个泛型类型，分别是输入类型和输出类型 ...

---------------


## 偏函数(Partial Function)的定义 ##

普通函数的定义中，有指定的输入类型，所以可以接受任意该类型下的值。比如一个(Int) => String的函数可以接收任意Int值，并返回一个字符串。

偏函数的定义中，也有指定的输入类型，但是偏函数不接受所有该类型下的值。比诶如(Int) => String的偏函数不能接收所有Int值为输入。

Scala的偏函数支持2个泛型类型，分别是输入类型和输出类型。

Scala中的偏函数数据类型为**PartialFunction**。

## 偏函数例子 ##

输入类型为Int，输出类型为String的偏函数例子:

	def one:PartialFunction[Int, String] = {
    	case 1 => "one"
    }
    
    one(1) // "one"
    one(2) // 报错
    
	def two: PartialFunction[Int, String] = {
    	case 2 => "two"
    }
    
    two(1) // 报错
    two(2) // "two"
    
    def wildcard: PartialFunction[Int, String] = {
    	case _ => "something else"
    }
    
    wildcard(1) // "something else"
    wildcard(2) // "something else"
    

由于偏函数只会接收部分参数，所以可以使用 "orElse" 方法进行组合：

	val partial = one orElse two orElse wildcard
    
    partial(1) // "one"
    partial(2) // "two"
    partial(3) // "something else"
    

## 偏函数原理 ##

PartialFunction是一个trait，继承了scala.Function1这个trait。

orElse就是PartialFunction内部的1个方法，PartialFunction还有其他一些方法，比如isDefinedAt， andThen，applyOrElse等方法。

isDefinedAt：判断偏函数是否对参数中的参数有效

	one.isDefinedAt(1) // true
    one.isDefinedAt(2) // false

andThen：相当于方法的连续调用，比如g(f(x))

orElse: 本文已经分析过

applyOrElse：接收2个参数，第一个是调用的参数，第二个是个回调函数。如果第一个调用的参数匹配，返回匹配的值，否则调用回调函数

	one.applyOrElse(2, { num:Int => "two" }) // "two"
    


当我们使用PartialFunction类型定义一个偏函数的时候，scala会被自动转换：

	def int2Char: PartialFunction[Int, Char] = {
    	case 1 => 'a'
        case 3 => 'c'
    }
    
会被scala转换为：

	val int2Char = new PartialFunction[Int, Char] {
    	def apply(i: Int) = {
        	case 1 => 'a'
            case 3 => 'c'
        }
        def isDefinedAt(i: Int): Boolean = i match {
        	case 1 => true
            case 3 => true
            case _ => false
        }
    }

## 部分应用(Partial application) ##

部分应用表示一个函数的调用并不需要全部的参数，可以使用 "_" 这个特殊符号代表一个参数，使用部分应用之后函数的返回值是函数，而不是具体的值。

	def add(x: Int, y: Int) = x + y
    
    val addOne = add(1, _: Int)
    
    addOne(4) // 5
    addOne(6) // 7

部分应用的参数可以应用到参数中的任意参数。
	






    
    
    

