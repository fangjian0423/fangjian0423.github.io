title: Java线程状态分析
date: 2016-06-04 23:53:35
tags:
- java
categories:
- java

--------------

Java线程的生命周期中，存在几种状态。在Thread类里有一个枚举类型State，定义了线程的几种状态，分别有：

1. NEW: 线程创建之后，但是还没有启动(not yet started)。这时候它的状态就是NEW
2. RUNNABLE: 正在Java虚拟机下跑任务的线程的状态。在RUNNABLE状态下的线程可能会处于等待状态， 因为它正在等待一些系统资源的释放，比如IO
3. BLOCKED: 阻塞状态，等待锁的释放，比如线程A进入了一个synchronized方法，线程B也想进入这个方法，但是这个方法的锁已经被线程A获取了，这个时候线程B就处于BLOCKED状态
4. WAITING: 等待状态，处于等待状态的线程是由于执行了3个方法中的任意方法。 1. Object的wait方法，并且没有使用timeout参数; 2. Thread的join方法，没有使用timeout参数 3. LockSupport的park方法。  处于waiting状态的线程会等待另外一个线程处理特殊的行为。 再举个例子，如果一个线程调用了一个对象的wait方法，那么这个线程就会处于waiting状态直到另外一个线程调用这个对象的notify或者notifyAll方法后才会解除这个状态
5. TIMED_WAITING: 有等待时间的等待状态，比如调用了以下几个方法中的任意方法，并且指定了等待时间，线程就会处于这个状态。 1. Thread.sleep方法 2. Object的wait方法，带有时间 3. Thread.join方法，带有时间 4. LockSupport的parkNanos方法，带有时间 5. LockSupport的parkUntil方法，带有时间
6. TERMINATED: 线程中止的状态，这个线程已经完整地执行了它的任务

<!--more-->


下面通过几个例子再次说明一下在什么情况下，线程会处于这几种状态：

## NEW状态

NEW状态比较简单，实例化一个线程之后，并且这个线程没有开始执行，这个时候的状态就是NEW：

	Thread thread = new Thread();
    System.out.println(thread.getState()); // NEW

## RUNNABLE状态

正在运行的状态。

	Thread thread = new Thread(new Runnable() {
        @Override
        public void run() {
            for(int i = 0; i < Integer.MAX_VALUE; i ++) {
                System.out.println(i);
            }
        }
    }, "RUNNABLE-Thread");
    thread.start();
    
使用jstack查看线程状态：
	
    "RUNNABLE-Thread" #10 prio=5 os_prio=31 tid=0x00007f8e04981000 nid=0x4f03 runnable [0x000070000124c000]
       java.lang.Thread.State: RUNNABLE
      at java.io.FileOutputStream.writeBytes(Native Method)
      at java.io.FileOutputStream.write(FileOutputStream.java:315)
      at java.io.BufferedOutputStream.flushBuffer(BufferedOutputStream.java:82)
      at java.io.BufferedOutputStream.flush(BufferedOutputStream.java:140)
      - locked <0x000000079764cc50> (a java.io.BufferedOutputStream)
      at java.io.PrintStream.write(PrintStream.java:482)
      - locked <0x0000000797604dc0> (a java.io.PrintStream)
      at sun.nio.cs.StreamEncoder.writeBytes(StreamEncoder.java:221)
      at sun.nio.cs.StreamEncoder.implFlushBuffer(StreamEncoder.java:291)
      at sun.nio.cs.StreamEncoder.flushBuffer(StreamEncoder.java:104)
      - locked <0x0000000797604d78> (a java.io.OutputStreamWriter)
      at java.io.OutputStreamWriter.flushBuffer(OutputStreamWriter.java:185)
      at java.io.PrintStream.write(PrintStream.java:527)
      - eliminated <0x0000000797604dc0> (a java.io.PrintStream)
      at java.io.PrintStream.print(PrintStream.java:597)
      at java.io.PrintStream.println(PrintStream.java:736)
      - locked <0x0000000797604dc0> (a java.io.PrintStream)
      at study.thread.ThreadStateTest$1.run(ThreadStateTest.java:23)
      at java.lang.Thread.run(Thread.java:745)
      
      
## BLOCKED状态

线程A和线程B都需要持有lock对象的锁才能调用方法。如果线程A持有锁，那么线程B处于BLOCKED状态；如果线程B持有锁，那么线程A处于BLOCKED状态。例子中使用Thread.sleep方法主要是用于调试方便：

	final Object lock = new Object();
    Thread threadA = new Thread(new Runnable() {
        @Override
        public void run() {
            synchronized (lock) {
                System.out.println(Thread.currentThread().getName() + " invoke");
                try {
                    Thread.sleep(20000l);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }, "BLOCKED-Thread-A");
    Thread threadB = new Thread(new Runnable() {
        @Override
        public void run() {
            synchronized (lock) {
                System.out.println(Thread.currentThread().getName() + " invoke");
                try {
                    Thread.sleep(20000l);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }, "BLOCKED-Thread-B");
    threadA.start();
    threadB.start();
    
使用jstack查看线程状态。由于线程A先执行，线程B后执行，而且线程A执行后调用了Thread.sleep方法，所以线程A会处于TIMED_WAITING状态，线程B处于BLOCKED状态：
      
    "BLOCKED-Thread-B" #11 prio=5 os_prio=31 tid=0x00007fa7db8ff000 nid=0x5103 waiting for monitor entry [0x000070000134f000]
       java.lang.Thread.State: BLOCKED (on object monitor)
      at study.thread.ThreadStateTest$3.run(ThreadStateTest.java:50)
      - waiting to lock <0x0000000795a03bf8> (a java.lang.Object)
      at java.lang.Thread.run(Thread.java:745)

    "BLOCKED-Thread-A" #10 prio=5 os_prio=31 tid=0x00007fa7db15a000 nid=0x4f03 waiting on condition [0x000070000124c000]
       java.lang.Thread.State: TIMED_WAITING (sleeping)
      at java.lang.Thread.sleep(Native Method)
      at study.thread.ThreadStateTest$2.run(ThreadStateTest.java:39)
      - locked <0x0000000795a03bf8> (a java.lang.Object)
      at java.lang.Thread.run(Thread.java:745)
      

## WAITING状态

Object的wait方法、Thread的join方法以及Conditon的await方法都会产生WAITING状态。

1.没有时间参数的Object的wait方法

	final Object lock = new Object();
    Thread threadA = new Thread(new Runnable() {
        @Override
        public void run() {
            synchronized (lock) {
                try {
                    lock.wait();
                    System.out.println("wait over");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }, "WAITING-Thread-A");
    Thread threadB = new Thread(new Runnable() {
        @Override
        public void run() {
            synchronized (lock) {
                try {
                    Thread.sleep(20000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                lock.notifyAll();
            }
        }
    }, "WAITING-Thread-B");
    threadA.start();
    threadB.start();
    
WAITING-Thread-A调用了lock的wait，处于WAITING状态：
    
    "WAITING-Thread-B" #11 prio=5 os_prio=31 tid=0x00007f8de992d800 nid=0x5103 waiting on condition [0x000070000134f000]
       java.lang.Thread.State: TIMED_WAITING (sleeping)
      at java.lang.Thread.sleep(Native Method)
      at study.thread.ThreadStateTest$5.run(ThreadStateTest.java:84)
      - locked <0x0000000795a03e40> (a java.lang.Object)
      at java.lang.Thread.run(Thread.java:745)

    "WAITING-Thread-A" #10 prio=5 os_prio=31 tid=0x00007f8dea193000 nid=0x4f03 in Object.wait() [0x000070000124c000]
       java.lang.Thread.State: WAITING (on object monitor)
      at java.lang.Object.wait(Native Method)
      - waiting on <0x0000000795a03e40> (a java.lang.Object)
      at java.lang.Object.wait(Object.java:502)
      at study.thread.ThreadStateTest$4.run(ThreadStateTest.java:71)
      - locked <0x0000000795a03e40> (a java.lang.Object)
      at java.lang.Thread.run(Thread.java:745)

2.Thread的join方法

	Thread threadA = new Thread(new Runnable() {
        @Override
        public void run() {
            try {
                Thread.sleep(20000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("Thread-A over");
        }
    }, "WAITING-Thread-A");
    threadA.start();
    try {
        threadA.join();
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    
主线程main处于WAITING状态：
    
    "WAITING-Thread-A" #10 prio=5 os_prio=31 tid=0x00007fd2d5100000 nid=0x4e03 waiting on condition [0x000070000124c000]
       java.lang.Thread.State: TIMED_WAITING (sleeping)
      at java.lang.Thread.sleep(Native Method)
      at study.thread.ThreadStateTest$6.run(ThreadStateTest.java:103)
      at java.lang.Thread.run(Thread.java:745)
    "main" #1 prio=5 os_prio=31 tid=0x00007fd2d3815000 nid=0x1003 in Object.wait() [0x0000700000182000]
       java.lang.Thread.State: WAITING (on object monitor)
      at java.lang.Object.wait(Native Method)
      - waiting on <0x0000000795a03ec0> (a java.lang.Thread)
      at java.lang.Thread.join(Thread.java:1245)
      - locked <0x0000000795a03ec0> (a java.lang.Thread)
      at java.lang.Thread.join(Thread.java:1319)
      at study.thread.ThreadStateTest.WAITING_join(ThreadStateTest.java:118)
      at study.thread.ThreadStateTest.main(ThreadStateTest.java:13)
      at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
      at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
      at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
      at java.lang.reflect.Method.invoke(Method.java:483)
      at com.intellij.rt.execution.application.AppMain.main(AppMain.java:140)

3.没有时间参数的Condition的await方法

Condition的await方法跟Obejct的wait方法原理是一样的，故也是WAITING状态
 
## TIMED_WAITING状态

TIMED_WAITING状态跟TIMEING状态类似，是一个有等待时间的等待状态，不会一直等待下去。

最简单的TIMED_WAITING状态例子就是Thread的sleep方法：

	Thread threadA = new Thread(new Runnable() {
        @Override
        public void run() {
            try {
                Thread.sleep(20000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("Thread-A over");
        }
    }, "WAITING-Thread-A");
    threadA.start();
    try {
        Thread.sleep(5000);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    System.out.println(threadA.getState()); // TIMED_WAITING
    
或者是Object的wait方法带有时间参数、Thread的join方法带有时间参数也会让线程的状态处于TIMED_WAITING状态。


## TERMINATED

线程终止的状态，线程执行完成，结束生命周期。

	Thread threadA = new Thread();
    threadA.start();
    try {
        Thread.sleep(5000l);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    System.out.println(threadA.getState()); // TERMINATED
     

## 总结
      
了解线程的状态可以分析一些问题。

比如线程处于BLOCKED状态，这个时候可以分析一下是不是lock加锁的时候忘记释放了，或者释放的时机不对。导致另外的线程一直处于BLOCKED状态。

比如线程处于WAITING状态，这个时候可以分析一下notifyAll或者signalAll方法的调用时机是否不对。


java自带的jstack工具可以分析查看线程的状态、优先级、描述等具体信息。

	
