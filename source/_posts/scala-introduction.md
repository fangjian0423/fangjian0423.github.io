title: Scala基础
date: 2015-05-31 09:49:50
tags:
- scala
- jvm
categories:
- scala
description: Scala是一门可以运行的jvm上的动态语言，其他可以运行在jvm上的动态语言还有groovy,clojure等 ...

---------------
## Scala简介 ##

Scala是一门可以运行的jvm上的动态语言，其他可以运行在jvm上的动态语言还有groovy,clojure等。

Scala是一种面向对象编程语言，同时又无缝结合了命令式和函数式的编程风格。

## 变量的定义以及赋值 ##

使用var或val定义变量。 var表示这个变量可变，val则表示这个变量不可变(类似java中的final关键字)。

	var a = 1; a = 2
    val a = 1; a = 1 // 报错，a不可变
    
## 基础类型 ##

scala有8种基础类型，分别是Byte, Short, Int, Long, Float, Double, Boolean, Char。

	var a: Int = 1 // 指定变量a的类型为Int，不指定的话scala会自动根据上下文得出变量的类型
    var str = "a test string"
	var ten = "10".toInt // string类型有toInt方法可以转换成Int类型
   
## 函数的定义 ##

函数的定义语法如下：

	def 函数名(参数名: 参数类型): 函数返回类型 = 函数体
    
1个简单的函数:

	def scalaFunc(name: String): Unit = println("welcome to use scala function: " + name)
    
   	// 函数调用
    scalaFunc("format") // 打印出 welcome to use scala cuntion: format
    
## if语法 ##

if语法跟其他语言类似

	if(some condition) value1 else value2
    
## for语法 ##

基本使用：

	val range = 1 to 10
    // 打印出1-10数字
    for(index <- num) {
    	println(index)
    }
    
在for循环中使用yield关键字可返回循环中的各个值并组成一个新的集合

	val range = 1 to 10
    for(index <- range) yield index // 得到一个1-10的新的集合
    
    
for循环中可以嵌入其他语法：

打印出在1-10中所有的偶数:

	for(index <- 1 to 10) {
    	if（index % 2 == 0) println(index)
    }
    
由于scala可以在for循环中嵌入其他语法，所以可以简化：

	for(index <- 1 to 10; if(index % 2 == 0)) println(index)
    
## List和Array的简单使用 ##

List和Array都支持类型参数化，也就是泛型。

Array的使用：

	val array = new Array[String](3)
    array(0) = "format01"
    array(1) = "format02"
    

List的使用：

	val list = List[Int](1, 2, 3)
    list ++ List(4, 5, 6) // List(1, 2, 3, 4, 5, 6)
    1 :: list // list前方加入元素。 List(1, 1, 2, 3)
    list :+ 1 // list后方加入元素。 List(1, 2, 3, 1)
    
