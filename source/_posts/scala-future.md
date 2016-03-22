title: Scala并发的Future
date: 2015-08-14 20:22:33
tags:
- scala
- concurrent
categories:
- scala
description: Future这个概念其实java里也已经有了， 表示一个未来的意思。 某线程执行一项操作，这个操作有延迟的话，Future会提供一系列方法来处理这个线程过程，可取消，可操作完成... 

---------------

Future这个概念其实java里也已经有了， 表示一个未来的意思。 某线程执行一项操作，这个操作有延迟的话，Future会提供一系列方法来处理这个线程过程，可取消，可操作完成后执行其他操作等等。
     
使用Future是非阻塞的，在Future中可以使用回调函数可以避免阻塞操作。 Scala在Future中提供了flatMap，foreach，filter等方法。

## Future概念 ##

Future的状态：

1.未完成：线程操作还未结束
2.已完成：操作操作完成，并且有返回值或者有异常。 当一个Future完成的时候，它就变成了一个不可变对象，永远不会被重写

构造Future最简单的方法是使用Future这个object提供的apply方法：

	Future {
    	...
    }

来看一个最简单的Future操作， 计算和：

	import scala.concurrent.ExecutionContext.Implicits.global
	import scala.concurrent.duration.Duration
	import scala.concurrent.{Await, Future}

    val sumFuture = Future[Int] {
      var sum = 0
      for(i <- Range(1,100000)) sum = sum + i
      sum
    }

    sumFuture.onSuccess {
      case sum => println(sum)
    }

    Await.result(sumFuture, Duration.Inf)

Await的作用是阻断Future等待Future的执行结果。 import scala.concurrent.ExecutionContext.Implicits.global 这个global表示一个ExecutionContext，类似线程池，所有线程的提交都得交给ExecutionContext。如果没有import这个global对象，那么执行的时候会报错。

文本文件中找关键字的例子，读io文件的时候如果文件很大，肯定会阻塞。使用Future完成，可以更有效率，等关键字索引找到的时候再去拿数据。这段时间完成可以去做其他事情：

	val keywordIndex = Future[Int] {
      val source = scala.io.Source.fromFile("intro.txx")
      source.toSeq.indexOfSlice("format")
    }
    
## 回调 ##

Future提供了3种Callback，分别是 onComplete，onFailure，onSuccess。

onComplete回调表示Future执行完毕了。需要1个Try[T] => U类型的参数，如果执行成功且没发生一次，那么匹配Success类型，否则匹配Failure类型。

onComplete例子：

	val calFuture = Future[Int] {
      val a = 1 / 1
      a
    }

    calFuture.onComplete {
      case Success(result) => println(result)
      case Failure(e) => println("error: " + e.getMessage)
    }

onFailure回调表示Future已经执行完成，但是出错了。

	val errorFuture = Future[Unit] {
      1 / 0
    }

    errorFuture.onFailure {
      case e => println(e.getMessage)
    }

onSuccess回调表示Future已经执行完成，而且执行成功了。

	val successFuture = Future[Int] {
      1000
    }

    successFuture.onSuccess {
      case num => println(num)
    }

## Future的组合 ##

使用map方法可以组合Future：

	val firstValFuture = Future[Int] {
      1
    }

    val secondFuture = firstValFuture.map { num =>
      println("firstFuture: " + num)
      num + "111"
    }

    secondFuture.onSuccess {
      case result => println(result)
    }

flapMap方法也可以组合Future，map方法和flatMap方法唯一的区别是flatMap内部需要返回Future，而map不是。

	val firstValFuture = Future[Int] {
      1
    }

    val secondFuture = firstValFuture.flatMap { num =>
      println("firstFuture: " + num)
      Future {
        num + "111"
      }
    }

    secondFuture.onSuccess {
      case result => println(result)
    }

其他资料参考[Scala的官方文档](http://docs.scala-lang.org/overviews/core/futures.html)即可。
