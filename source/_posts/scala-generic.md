title: Scala泛型
date: 2015-06-07 18:17:21
tags:
- scala
- jvm
categories:
- scala
description: scala中的泛型称为类型参数化(type parameterlization)。语法跟java不一样，使用"[]"表示类型 ...

---------------

## 简单回顾泛型 ##

java中可使用泛型进行编程，一个简单的泛型例子如下：

	List<String> strList = new ArrayList<String>();
    strList.add("one");
    strList.add("two");
    strList.add("three");
    
    String one = strList.get(0); // 泛型拿数据不必进行类型转换，不使用泛型的话需要对类型进行转换
    
## scala的泛型 ##

scala中的泛型称为类型参数化(type parameterlization)。语法跟java不一样，使用"[]"表示类型。

一个使用类型参数化的函数：

	def position[A](xs: List[A], value: A): Int = {
    	xs.indexOf(value)
    }
    
    position(List(1,2,3), 1) // 0
    position(List("one", "two", "three"), "two") // 1
    

稍微复杂点的类型参数化，实现一个map函数，需要一个List和一个函数作为参数：

普通的map方法：

	List(1,2,3) map { _ * 2 }  // List[Int] = List(2,4,6)
    List(1,2,3) map { _ + "2" }  // List[String] = List(12, 22, 32)
    
使用泛型实现的map方法：

	def map[A,B](list:List[A], func: A => B) = list.map(func)
    
    map(List(1,2,3), { num: Int => num + "2" }) // List[String] = List(12, 22, 32)
    map(List(1,2,3), { num: Int => num * 2 }) // List[Int] = List(2, 4, 6) 
    
## 上限和下限 ##

scala跟java一样，也提供了上限(upper bounds)和下限(lower bounds)功能。

### 上限(upper bounds) ###

java中上限的使用如下：

	<T extends Object>
    
    通配符形式
    <? extends Object>

scala写法：

	[T <: AnyRef]
    
    通配符形式
    [_ <: AnyRef]


上限的一些例子：

	public void upperBound(List<? extends Number> list) {
        Object obj = list.get(0); // Number是Object的子类，使用Object可以代替Number。
        Number num = list.get(0);
        Integet i = list.get(0); // compile error
        list.add(new Integer(1)); // compile error
    }
    
上限做参数，**set的话不能确定具体的类型**，所以会报编译错误。get的话得到的结果的类型的下限为参数的上限。相当于使用了上限参数的话，该参数就变成了**只读参数**，类似生产者，只提供数据。

scala版本：

	def upperBound[A <: Animal](list: ListBuffer[A]): Unit = { 
    	list += new Animal("123") // compile error
        val obj: AnyRef = list(0) // ok
        val a: Animal = list(0) // ok
        val a: Cat = list(0) // compile error
    } 
    
这里使用ListBuffer作为集合，ListBuffer的+=方法会在列表内部添加数据，不会产生一个新的List。如果使用List的话，:+操作符会在新生成的List中自动得到符合所有元素的类型。

	List[Animal](new Cat()) :+ 1 // List[Any] = List(Cat@3f23a076, 1)
    
生成新的List会自动根据上下文得到新的泛型类型List[AnyRef]。
    

### 下限(lower bounds) ###
    
    <T super MyClass>
    
    通配符形式
    <? super MyClass>
    
scala写法：

	[T >: MyClass]
    
    通配符形式
    [_ >: MyClass]

下限的一些例子：

	public static void lowerBound(List<? super Number> l) {
        l.add(new Integer(1));
        l.add(new Float(2));
        Object obj = l.get(0);
        Number num = l.get(0); // compile error
    }
    
下限做参数，get方法只能用最宽泛的类型来获取数据，相当于get只提供了数据最小级别的访问权限。类似消费者，主要用来消费数据。

scala版本：

	def lowerBound[A >: Animal](list: ListBuffer[A]): Unit = { 
    	list += new Animal() // ok
        list += new Cat() // ok
        val obj: Any = list(0) // ok
        val obj: Animal = list(0) // compile error
    }
    
## 协变和逆变 ##

协变(covariance)：对于一个带类型参数的类型，比如List[T]，如果对A及其子类型B，满足List[B]也符合List[A]的子类型，那么就称为协变，用加号表示。比如 MyType[+A]

逆变(contravariance)：如果List[A]是List[B]的子类型，用减号表示。比如MyType[+B]

如果一个类型支持协变或逆变，则称这个类型为可变的(variance)。否则称为不可变的(invariant)。

在java里，泛型类型都是不可变的，比如List&lt;String&gt;并不是List&lt;Object&gt;的子类。


	trait A[T]
    class C[T] extends A[T]
    class Parent; class Child extends Parent
    val c:C[Parent] = new C[Parent] // ok
    val c:C[Parent] = new C[Child]; // Child <: Parent, but class C is invariant in type T.
    
### 协变 ###

上面的例子提示已经很明确了。类C是不可变的，改成协变即可。

	trait A[+T]
    class C[+T] extends A[T]
    class Parent; class Child extends Parent
    val c: C[Parent] = new C[Parent] // ok
	val c: C[Parent] = new C[Child]  // ok
    

scala中List就是一个协变类。

	val list:List[Parent] = List[Child](new Child())
    
### 逆变 ###

逆变概念与协变相反。

	trait A[-T]
    class C[-T] extends A[T]
    class Parent; class Child extends Parent
    val c: C[Parent] = new C[Parent] // ok
	val c: C[Child] = new C[Parent]  // ok


### 协变逆变注意点 ###

逆变协变并不会被继承，父类声明为逆变或协变，子类如果想要保持，任需要声明：

	trait A[+T]
    class C[T] extends A[T] // C是不可变的，因为它不是逆变或协变。
    class D[+T] extends A[T] // D是可变的，是协变
    class E[-T] extends A[T] // E是可变的，是逆变


