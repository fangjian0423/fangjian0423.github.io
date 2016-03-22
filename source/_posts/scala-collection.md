title: Scala集合相关知识
date: 2015-06-20 01:59:18
tags:
- scala
- jvm
categories:
- scala
description: Scala各个集合类都会有两种类型，分别是可变的(mutable)和不可变的(immutable)

---------------


Scala各个集合类都会有两种类型，分别是可变的(mutable)和不可变的(immutable)。

比如列表集合List，不可变的类型是List，可变的类型为ListBuffer：

	// 不可变集合，集合添加元素后会产生一个新的集合
	val immutableList = scala.collection.immutable.List(1, 2, 3)
	immutableList(1) = 3 // =号底层会调用update方法，immutable.List没有这个update方法
    
    val mutableList = scala.collection.mutable.ListBuffer(1, 2, 3)
    mutableList(1) = 3 // ListBuffer(1, 3, 3), ListBuffer有update方法


Scala集合中3个主要的集合类的继承结构如下：

![](http://7x2wh6.com1.z0.glb.clouddn.com/scala_collection1.png)


Traversable是所有集合类的父类，它会提供一些很基础有用的方法，如下：

1.size: 返回集合元素的个数
	
    List(1,2,3).size // 3
    
2.++方法：合并两个集合

	List(1, 2, 3) ++ List(4, 5, 6) // List(1, 2, 3, 4, 5, 6)
    
3.map方法：使用函数作用于集合中的各个元素，生成新的集合

	List(1,2,3).map { _ + 1 } // List(2, 3, 4)
    
4.flatMap方法：跟map方法类似，只不过flatMap整合的是集合，会生成新的集合

	List(1, 2, 3).flatMap { x => List(x + 1, x + 2) } List(2, 3, 3, 4, 4, 5)
    
5.filter方法：使用函数过滤出符合条件的元素，生成新的集合

	List(1, 2, 3).filter { _ < 2 }  // List(1)

6.find方法：找出第一个符合条件的符合，并得到这个元素。返回值的类型是个Option

	List(1, 2, 3).find { _ < 3 } // Option[Int] = Some(1)
    List(1, 2, 3).find { _ > 3 } // Option[Int] = None
    
7./:方法，/:即foldLeft。foleLeft方法类似reduce方法，只不过多了一个初始值，reduce是从左边开始进行的

	List(1, 2, 3).foldLeft(10) { _ + _ } // 16
    
	(10 /: List(1, 2, 3)) { _ + _ } // 16

8.:\方法，:\即foldRight。跟foldLeft相反，reduce是从右边开始进行的

	List(1, 2, 3).foldRight(10) { _ + _ } // 16
    
    (List(1, 2, 3) :\ 10) { _ + _ } // 16

9.head方法：集合的第一个元素

	List(1, 2, 3).head // 1

10.tail方法：除了集合的第一个元素之外的其他元素的集合

	List(1, 2, 3).tail // List(2, 3)

11.mkString方法：使用字符串连接集合中的各个元素

	List(1, 2, 3).mkString("=") // 1=2=3
    List(1, 2, 3).mkString("===") // 1===2===3

	

## Set集合 ##

Set也分两种集合，可变的Set和不可变的Set。

不可变的Set的全名是scala.collection.immutable.Set
可变的Set全名为scala.collection.mutable.Set

Set常用的一些方法如下：

1.contains方法，查看元素是否存在
	
	Set(1,2,3).contains(2) // true
    Set(1,2,3).contains(4) // false
    // 可以直接调用set对象，相当于是contains方法
    Set(1,2,3)(4) // false， 相当于 val s = Set(1,2,3), a(4) false
    
2.++方法，合并2个集合

	Set(1,2,3) ++ Set(3,4,5) // Set(1,2,3,4,5)
    
3.&方法，交集

	Set(1,2,3) & Set(2,3,4) // Set(2,3)
    
4.|方法，并集

	Set(1,2,3) | Set(2,3,4) // Set(1,2,3,4)
    
5.&~方法，差集

	Set(1,2,3) &~ Set(2,3,4) // Set(1)
    
6.+=方法，集合内部添加元素，只能在mutable.Set中使用

	val set1 = scala.collection.immutable.Set(1)
    set1 += 2 // 报错
    val set2 = scala.collection.mutable.Set(1)
    set2 += 2 // set2: Set(1,2)
    
7.update方法，添加，删除元素，只能在mutable.Set中使用。 用法： set.update(value, true | false)

	val set = scala.collection.mutable.Set(1,2,3)
    set.update(4, true) // set: Set(1,2,3,4)
    set.update(10, true) // set: Set(1,2,3,10,4)
    set.update(10, false) // set: Set(1,2,3,4)

8.clear方法，只能在mutable.Set中使用，清除集合中的所有元素


SortedSet: 跟Set一样，唯一的区别是SortedSet会记录插入的顺序。


## Map和Tuple ##

Scala中Map中的每一项都是一个scala.Tuple2，也就是1个带有2个变量的元组。

Map可以这样定义：

	val map1 = Map(1 -> "one", 2 -> "two")
    val map2 = Map((1, "one"), (2, "two"))
    
获得对应键的值：

	map1(1) // "one"
	map1(3) // 报错。没有对应的值的话会报错    

使用Map的get方法可以避免这个问题，get方法返回的是Option，如果有值，返回Some，否则返回None：

	map1.get(1) // Some
    map1.get(3) // None
    map1.get(1).getOrElse(999) // "one"
    map1.get(3).getOrElse(999) // 999
    
    
Map常用的方法如下：

1.getOrElse(k, d)，获得键值为k的对应的值，不存在的话使用默认值d

	map1.getOrElse(3, "None") // "None"

2.+方法， map + (k -> v)，给map添加一个新的项，返回一个新的map
    
	map1 + (3 -> "three") // 返回一个数3个项的新的map
    
3.++方法，合并2个map

	Map(1 -> "one") ++ Map(2 -> "two")
    
4.filterByKey，根据key进行过滤

	map1.filterByKey { key => key == 1 }
        
5.mapValues，遍历所有的value值

	map1.mapValues { value => println(value) }
    
6.update方法，修改map中的数据。只适用于mutable.Map

	map1.update(1, "oneone")
    
7.clear方法，清空map数据。只适用于mutable.Map

Map中的每一个都是一个Tuple2，Tuple2取第一个元素可以使用_1获得，第二个元素使用_2获得。

	map1.foreach { item => println(item._1 + "," + item._2) }





