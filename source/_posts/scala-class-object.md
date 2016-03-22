title: Scala类和对象
date: 2015-05-31 22:02:14
tags:
- scala
- jvm
categories:
- scala
description: 介绍Scala类和对象

---------------

## 类的定义 ##

	class Person(val name:String, val age: Int)
    
## 构造函数中的参数修饰符 ##

### 用val修饰，就是类的定义中的那种方式 ###

使用val修饰，属性会有对应的getter方法。

	val p = new Person("format", 99)
    p.name // format
    p.name = "new name" // 报错，用val修饰的变量不可变
    
### 用var修饰 ###

使用var修饰，属性会有对应的getter方法。同时还会有setter方法。

	class Person(var name:String, var age: Int)
    val p = new Person("format", 99)
    p.name // format
    p.name = "new name"
    p.name // new name
    p.age = 100
    
### 不用任何修饰符 ###

不适用任何修饰符代表这个这些属性被当做私有的实例值(private instance values)，在类的外部不能被访问。

	class Person(name:String, age: Int)
    val p = new Person("format", 99)
    p.name // 报错，error: value name is not a member of Person
    p.name = "new name" // 报错，error: value name is not a member of Person
    
### 使用private修饰符 ###

private修饰的属性不会有对应的生成器(accessors)，外部无法访问。

	class Person(private var name:String, private var age: Int)
	val p = new Person("format", 99)
    p.name // error: variable name in class Person cannot be accessed in Person

### 手动添加getter和setter生成器 ###

	class Person(name:String, private var _age: Int) {
   		def age = _age // getter生成器
        def age_=(newAge: Int) = age = newAge // setter生成器
    }
    val p = new Person("format", 99)
    p.age // 99
    p.age = 1
    p.age // 1
    
    p.age = 3会被scala解析成p.age_=(3)

### 构造函数的重载 ###

	class Person(var name:String, var age: Int) {
    	def this() = this("format", 11)
    }
    val p = new Person()
    p.name // format
    p.age // 11
     
## 单例对象的定义 ##

### 单例对象 ###
	
scala不提供static关键字，也就是说没有静态变量，静态类，静态方法等概念了，取而代之的是使用object关键字，也就是单例对象。

	object Switch {
    	def open: Boolean = true
    }
    
    Switch.open // true
    
### apply方法 ###

对象的apply方法比较特殊。

	object Person {
    	def apply(name: String, age: Int) = new Person(name, age)
    }
    
	val p = Person("format", 99)
    p.name // format
    p.age // 99
    
apply方法的作用也就是可以让对象被直接调用，这里Person的apply方法返回的是一个Person类的实例，Person类也就是之前定义的一个类。
    
### unapply方法 ###

对象的unapply方法用于模式匹配(pattern match)。模式匹配的内容会在之后的文章中说明。
    
    object Person {
    	def unapply(p: Person): Option[(String, Int)] = Some((p.name, p.age))
    }
    
    val p = new Person("format", 99)
    p match {
    	case Person(name, age) => println(name + "," + age)
    }
	

## 样本类 (case class) ##

样本类的定义:
	
    case class Person(name: String, age: Int)
    
样本类会自定生成一些样本代码：

1. 会给所有的参数带上val前缀(可以手动改为var前缀，case class Person(var name: String, age: Int))
2. equals和hashCode方法会根据参数自动生成
3. toString方法也会自动生成，会带上类名和参数
4. 所有的case类都会有有一个copy方法
5. companion object会自动生成，且还会带有apply方法，apply方法的参数跟类的参数一样(companion object也就是对应类的单例对象，比如本文提到了Person类和Person对象)
6. companion object还会有unapply方法
7. 会有一个默认的序列化实现(a default implementation is provided for serialization)

以之前定义的Person样本类为例，验证一些样本代码：

	val p = Person("format", 99)  // case Person会生成一个带有apply的Person对象
    p.name // format
    p.age // 99
    p // Person(format, 99)
    p == Person("format", 99) // true
	p == Person("format", 9) // false
    val p2 = p.copy() // p2: Person(format, 99)
    p == p2 // true
    p.copy(name="newName") // Person(newName, 99)
    p.hashCode // 187596955
    p2.hashCode //187596955
    // 模式匹配，unapply方法
    p match {
    	case Person(name, age) => println(name + "," + age)
    } 
    val p = new Person("name", 99) // 不用单例对象，用new构造实例
