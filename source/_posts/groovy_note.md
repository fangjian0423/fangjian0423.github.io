title: groovy基本概念学习笔记
date: 2015-02-06 23:43:57
tags: [groovy,jvm]
description: 介绍一下groovy的基本语法
----------------

groovy是一种基于JVM的动态语言，能够与java代码很好地结合，可以使用Java语言编写的库。 

这里记录一下groovy的一些学习笔记。

1.groovy可以使用def定义一个变量，使用def其实就是表示这个对象的类型是Object：

    def num = 1
    def str = "format"
    def today = new Date()

而在java中，由于java是静态语言，因此java定义变量的时候必须指明变量的类型。


2.groovy中 "==" 符号相当于java中的 "equals" 方法， groovy中的 "is" 方法相当于java中的 "=="

    new Date() == new Date()   // true
    new Date().is(new Date())  // false

3.groovy中可以使用 "[]" 表示数组或链表， "[:]" 表示map。

    [1,2,3] // 表示包含一个ArrayList实例
    ["1":"a", "2":"b"] // 表示一个map

4.可以使用dump方法查看当前实例的具体信息

    ["1":"a", "2":"b"] // <java.util.LinkedHashMap@a0 head=1=a tail=2=b accessOrder=false table=[null, 1=a, 2=b, null] entrySet=[1=a, 2=b] size=2 modCount=2 threshold=3 loadFactor=0.75 keySet=null values=null>

5.groovy会默认import一些包。

    java.io.*
    java.lang.*
    java.math.BigDecimal
    java.math.BigInteger
    java.net.*
    java.util.*
    groovy.lang.*
    groovy.util.*

而java只会默认import    java.lang.*

6.java空对象调用方法会报错，因此需要加上逻辑判断一些非null对象，而groovy可以使用?.符号：

    instance?.method()

而在java中需要这么写：

    if(instance != null)
        instance.method()

7.groovy提供了一种叫做GString的概念。

    "${name}, nice to meet you!"

上面这段字符串就是个GString，字符串中的${name}会被自动解析成name变量所表示的值。

8.闭包

闭包应该是最大的区别了。 动态语言都有闭包这个概念，而静态语言没有。 

java遍历map的时候需要得到map的keySet，然后使用for遍历：

    Set<String> keySet = map.keySet();
    for(String str : keySet) {
        ...
    }

groovy中的闭包：

    map.each {
        key, value -> ....
    }  

list也可以使用闭包：

    list.eachWithIndex {
        value, index -> ...
    }

如果闭包内只有1个属性，那么这个属性可以不写，默认为it：

    [5,6,7].each {
        println it
    }

等于：

    [5,6,7].each {
        num -> println num
    }


9.范围range

groovy中可以使用1..3这个range，表示[1,2,3]。 1..3也可以写成 1..<4 


10.list实用方法

list的findAll方法，find方法，groupBy方法。

找出30岁的人，返回值也是一个List

    list.each {
      it.age == 30  // 注意是==。 如果用=的话不会报错，但是有时候会出现莫名其妙的错误，很难找。
    }

根据年龄分组，返回值是一个Map，key是age，也就是年龄，value是一个list

    list.groupBy {
      it.age
    }

11.日期实用方法

昨天和明天的获取：

    def now = new Date()
    def tomorrow = now + 1
    def yesterday = now - 1

根据时分秒设置时间：

    def now = new Date()
    now.set([hourOfDay: 1, minute: 30, second: 45])
