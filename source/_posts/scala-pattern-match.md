title: Scala模式匹配
date: 2015-06-04 01:03:27
tags:
- scala
- jvm
categories:
- scala
description: 模式匹配是一种scala中的一种函数式编程概念，跟java中的switch case类似 ...

---------------

## 模式匹配介绍 ##

模式匹配是一种scala中的一种函数式编程概念，跟java中的switch case类似。

java中的switch case比较局限，只支持基本类型和枚举类型；而且switch case如果case不到相应的值，除非在有写default，否则就不会执行。

模式匹配简单例子：

	1 match {
    	case 1 => println("yeah, it is 1")
    }
    
## 模式匹配的应用 ##

模式匹配可以匹配字符串，复杂值，类型，变量，常量，构造函数，元组。

变量匹配本文介绍的简单例子就可说明。

### 字符串匹配 ###

	val name = "format"
    name match {
    	case "format" => println("my name is format")
    }
    
### 类型匹配 ###
	
    def printType(obj: AnyRef) = {
    	case s:String => println("this is String")
        case l:List[_] => println("this is List")
        case a:Array[_] => println("this is array")
        case d:java.util.Date => println("this is Date")
    }
    
    printType("1") // this is String
    printType(List(1)) // this is List
    printType(Array(1)) // this is Array
    printType(new java.util.Date()) // this is Date
    printType(Map(1 -> 2)) // 报错。scala.MatchError: Map(1 -> 2) (of class scala.collection.immutable.Map$Map1)
    
### 构造函数匹配 ###

之前介绍scala的对象的[unapply方法](http://fangjian0423.github.io/2015/05/31/scala-class-object/#unapply方法)的时候提到了这点。 构造函数的模式匹配基于对象的unapply方法。

	case class Person(name: String, age: Int)
    val p = new Person("format", 99)
    // 打印 format,99
    p match {
    	case Person(name, age) => println(name + "," + age)
    }

### 元组匹配 ###

	// 打印1,2
	(1,2) match {
    	case (a,b) => println(a+","+b)
    }
    
    // 打印2
    (1,2) match {
    	case (1, b) => println(b)
    }
    
    // 打印found
    (1,2) match {
    	case (_, 2) => println("found")
    }

### 特殊的模式匹配 ###

#### 中缀操作符匹配(infix operation pattern) ####

直接看例子：

	// 得到List(1,2)
	List(1,2,3,4) match {
    	case a :: b :: other => List(a, b)
        case _ => List()
    }
    
a和b会被解析成1和2，other会被解析成List(3,4)。 :: 操作符是List的添加操作符。


#### 带有逻辑判断的模式匹配 ####

case里面可以添加部分逻辑代码：

	1 match {
    	case i if i < 10 => println("less than 10")
        case i if i >= 10 => println("more than 10")
    }
    
## 模式匹配要注意的几个地方 ##

1.没有匹配到的模式匹配会报错

	// 报错， 2没有匹配1
	2 match {
    	case 1 => println("get one")
    }

2.使用 "_" 代替没匹配到的情况

跟java里的switch case里的default语法一样。

	// error get num
	2 match {
    	case 1 => println("get one")
        case _ => println("error get num")
    }

