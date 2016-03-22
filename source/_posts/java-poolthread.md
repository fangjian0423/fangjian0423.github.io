title: java内置的线程池笔记
date: 2015-07-24 20:32:33
tags:
- java
- thread
categories:
- java
description: 写这篇博客的时候又刚好想起了当时自己实习的时候遇到的一个问题。1000个爬虫任务使用了多线程的处理方式，比如开5个线程处理这1000个任务 ...

---------------

目前的工作是接触大数据相关的内容，自己也缺少高并发的知识，刚好前几天看了flume的源码，里面也用到了各种线程池内容，刚好学习一下，做个笔记。

写这篇博客的时候又刚好想起了当时自己实习的时候遇到的一个问题。1000个爬虫任务使用了多线程的处理方式，比如开5个线程处理这1000个任务，每个线程分200个任务，然后各个线程处理那200个爬虫任务→_→，太笨了。其实更合理的方法是使用阻塞队列+线程池的方法。

## 线程池的使用 ##

Java把线程的调用封装成了一个Executor接口，Executor接口中定义了一个execute方法，用来提交线程的执行。

而ExecutorService接口是Executor接口的子接口，用来管理线程的执行。

ExecutorService的初始化可以使用Executors类的静态方法。

Executors类是Java提供的一个专门用于创建线程池的类。

Executors提供了很多方法用来初始化ExecutorService，可以初始化指定数目个线程的线城市或者单个线程的线程池。

比如构造一个10个线程的线程池，使用了guava的ThreadFactoryBuilder，guava的ThreadFactoryBuilder可以传入一个namFormat参数用户来表示线程的name，它内部会使用数字增量表示%d，比如一下的nameFormat，10个线程，名字分别是thread-call-runner-1，thread-call-runner-2 ... thread-call-runner-10:

	Executors.newFixedThreadPool(10, new ThreadFactoryBuilder().setNameFormat("thread-call-runner-%d").build());
    
    
ExecutorService线程池使用线程执行任务例子：

1.阻塞队列里有10个元素，初始化带有2个线程的线程池，跑2个线程分别去阻塞队列里取数据执行。

	@Test
    public void test01() throws Exception {
        ExecutorService es = Executors.newFixedThreadPool(2, new ThreadFactoryBuilder().setNameFormat("thread-call-runner-%d").build());
        final LinkedBlockingDeque<String> deque = new LinkedBlockingDeque<String>();
        for(int i = 1; i <= 10; i ++) {
            deque.add(i + "");
        }
        es.submit(new Runnable() {
            @Override
            public void run() {
                while(!deque.isEmpty()) {
                    System.out.println(deque.poll() + "-" + Thread.currentThread().getName());
                }
            }
        });
        es.submit(new Runnable() {
            @Override
            public void run() {
                while(!deque.isEmpty()) {
                    System.out.println(deque.poll() + "-" + Thread.currentThread().getName());
                }
            }
        });
        Thread.sleep(10000l);
    }
    
打印，2个线程都会执行：

	1-thread-call-runner-0
    2-thread-call-runner-0
    3-thread-call-runner-1
    4-thread-call-runner-0
    5-thread-call-runner-0
    6-thread-call-runner-1
    7-thread-call-runner-0
    8-thread-call-runner-1
    9-thread-call-runner-0
    10-thread-call-runner-0

2.执行Callable线程，Callable线程和Runnable线程的区别就是Callable的线程会有返回值，这个返回值是Future，未来的意思，而且这Future是个接口，提供了几个实用的方法，比如cancel, idDone, isCancelled, get等方法。

	@Test
    public void test02() throws Exception {
        ExecutorService es = Executors.newFixedThreadPool(2, new ThreadFactoryBuilder().setNameFormat("thread-call-runner-%d").build());
        final LinkedBlockingDeque<String> deque = new LinkedBlockingDeque<String>();
        for(int i = 1; i <= 500; i ++) {
            deque.add(i + "");
        }
        Future<String> result = es.submit(new Callable<String>() {
            @Override
            public String call() throws Exception {
                while (!deque.isEmpty()) {
                    System.out.println(deque.poll() + "-" + Thread.currentThread().getName());
                }
                return "done";
            }
        });
        System.out.println(result.isDone());
        // get方法会阻塞
        System.out.println(result.get());
        System.out.println("exec next");
    }
    
打印：

	先打印出几百个 数字-thread-call-runner-0
    然后打印出 isDone的结果， 是false
    Future的get是得到Callable线程执行完毕后的结果，该方法会阻塞，直到该Future对应的线程全部执行完才会继续执行下去。这个例子Callable线程执行完返回done，所以get方法也是返回done
    
	get方法还有个重载的方法，带有2个参数，第一个参数是一个long类型的数字，第二个参数是时间单位。result.get(1, TimeUnit.MILLISECONDS) 就表示1毫秒，等待这个Future的时间为1毫秒，如果时间1毫秒，那么这个get方法的调用会抛出java.util.concurrent.TimeoutException异常，并且线程内部的执行也会停止。注意，但是如果我们catch这个TimeoutException的话，那么线程里的代码还是会被执行完毕。
    
    try {
    	// catch TimeoutException的话线程里的代码还是会执行下去
        System.out.println(result.get(10, TimeUnit.MILLISECONDS));
    } catch (TimeoutException e) {
        e.printStackTrace();
    }
    
3.Future的cancel方法的使用

	@Test
    public void test03() throws Exception {
        ExecutorService es = Executors.newFixedThreadPool(2, new ThreadFactoryBuilder().setNameFormat("thread-call-runner-%d").build());
        final LinkedBlockingDeque<String> deque = new LinkedBlockingDeque<String>();
        for(int i = 1; i <= 5000; i ++) {
            deque.add(i + "");
        }
        Future<String> result = es.submit(new Callable<String>() {
            @Override
            public String call() throws Exception {
                while (!deque.isEmpty() && !Thread.currentThread().isInterrupted()) {
                    System.out.println(deque.poll() + "-" + Thread.currentThread().getName());
                }
                return "done";
            }
        });

        try {
            System.out.println(result.get(10, TimeUnit.MILLISECONDS));
        } catch (TimeoutException e) {
            System.out.println("cancel result: " + result.cancel(true));
			System.out.println("is cancelled: " + result.isCancelled());
        }
        Thread.sleep(2000l);
    }
    
打印：

	先打印出几百个 数字-thread-call-runner-0
    然后打印出
    cancel result: true
	is cancelled: true
    
    cancel方法用来取消线程的继续执行，它有个boolean类型的返回值，表示是否cancel成功。这里我们使用了get方法，10毫秒处理5000条数据，报了TimeoutException异常，catch之后对Future进行了cancel调用。注意，我们在Callable里执行的代码里加上了
    	
	!Thread.currentThread().isInterrupted()
    
    如果去掉了这个条件，那么还是会打印出5000条处理数据。cancel方法底层会去interrupted对应的线程，所以才需要加上这个条件的判断。

## 调度线程池的使用 ##

ScheduledExecutorService接口是ExecutorService接口的子类。

看名字也知道，ScheduledExecutorService是基于调度的线程池。

1.ScheduledExecutorService的schedule方法例子：

	@Test
    public void test04() throws Exception {
        ScheduledExecutorService ses = Executors.newScheduledThreadPool(2, new ThreadFactoryBuilder().setNameFormat("thread-schedule-runner-%d").build());
        Future<String> result = ses.schedule(new Callable<String>() {
            @Override
            public String call() throws Exception {
                System.out.println("exec task");
                return "ok";
            }
        }, 10, TimeUnit.SECONDS);
        Thread.sleep(15000);
    }

打印：

	执行10秒后打印出  exec task
    
2.cancel在schedule中的使用：

	@Test
    public void test05() throws Exception {
        ScheduledExecutorService ses = Executors.newScheduledThreadPool(2, new ThreadFactoryBuilder().setNameFormat("thread-schedule-runner-%d").build());
        Future<String> result = ses.schedule(new Callable<String>() {
            @Override
            public String call() throws Exception {
                System.out.println("exec task");
                try {
                    Thread.sleep(5000l);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("exec done, take 5 seconds");
                return "ok";
            }
        }, 10, TimeUnit.SECONDS);
        Thread.sleep(11000);
        result.cancel(true);
        Thread.sleep(10000);
    }
    
打印：

	先打印出exec task，然后抛出InterruptedException异常，异常被catch。接着打印出exec done, take 5 seconds
	因为Callable线程是10秒后执行的，线程会执行5秒，在11秒的时候会调用Future的cancel方法，会取消线程的时候，由于我们catch了异常，所以线程会执行完毕。
    
    注意一下，cancel方法有个boolean类型的参数mayInterruptIfRunning。上个例子中我们传入了true，所以会打断正在执行的线程，因此抛出了异常。如果我们传入false，线程正在执行，所以不会去打断它，因此会打印出exec task，然后再打印出exec done, take 5 seconds，并且没有异常抛出。

3.scheduleWithFixedDelay方法，定时器，每隔多少时间执行

	@Test
    public void test06() throws Exception {
        ScheduledExecutorService sec = Executors.newScheduledThreadPool(2, new ThreadFactoryBuilder().setNameFormat("thread-schedule-runner-%d").build());
        sec.scheduleWithFixedDelay(new Runnable() {
            @Override
            public void run() {
                System.out.println("exec");
            }
        }, 0, 5, TimeUnit.SECONDS);
        Thread.sleep(16000l);
    }
    
打印：

	打印出4次exec。
    scheduleWithFixedDelay有4个参数，第一个参数是个Runnable接口的实现类，第二个参数是首次执行线程的延迟时间，第三个参数是每隔多少时间再次执行线程时间，第四个参数是时间的单位。
    如果Runnable中执行的代码发生了异常并且没有被catch的话，那么发生异常之后，Runnable里的代码就不会再次执行。
    
4.scheduleAtFixedRate方法，scheduleAtFixedRate方法跟scheduleWithFixedDelay类似。唯一的区别是scheduleWithFixedDelay是在线程全部执行完毕之后开始计算时间的，而scheduleAtFixedRate是在线程开始执行的时候计算时间的，所以scheduleAtFixedRate有时会产生不是定时执行的感觉。

先看scheduleWithFixedDelay：

	@Test
    public void test07() throws Exception {
        ScheduledExecutorService sec = Executors.newScheduledThreadPool(2, new ThreadFactoryBuilder().setNameFormat("thread-schedule-runner-%d").build());
        sec.scheduleWithFixedDelay(new Runnable() {
            @Override
            public void run() {
                System.out.println("exec");
                try {
                    Thread.sleep(3000l);
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }, 0, 5, TimeUnit.SECONDS);
        Thread.sleep(17000l);
    }
    
打印：

	执行3次exec。Runnable每次执行3秒。第一次是在0秒执行，执行了3秒，第二次是在8秒，执行了3秒。第三次是在16秒执行
    
然后再看scheduleAtFixedRate：

	@Test
    public void test09() throws Exception {
        ScheduledExecutorService sec = Executors.newScheduledThreadPool(2, new ThreadFactoryBuilder().setNameFormat("thread-schedule-runner-%d").build());
        sec.scheduleAtFixedRate(new Runnable() {
            @Override
            public void run() {
                System.out.println("exec");
                try {
                    Thread.sleep(3000l);
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }, 0, 5, TimeUnit.SECONDS);
        Thread.sleep(16000l);
    }
    
打印：

	执行了4次exec，第一次在0秒执行，第二次在5秒，第三次是10秒，第四次在15秒执行


## 线程池的参数解释 ##


Executors类的静态方法中大致有这么几种创建不同类型的线程池：

1. newFixedThreadPool
2. newSingleThreadExecutor
3. newCachedThreadPool
4. newScheduledThreadPool
5. newSingleThreadScheduledExecutor


在分析这些方法之前，先解释一下线程池中所使用的一些配置参数。

比如实例化一个单线程的线程池。源码如下：

    public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }
    
ThreadPoolExecutor是线程池的具体实现类。它的构造函数接受以下这几个重要的参数：

1. corePoolSize：线程池基本数量
2. workQueue：用于保存等待执行的任务的阻塞队列。阻塞队列的类型可自己选择，阻塞有这几种类型，ArrayBlockingQueue(基于数组的有界阻塞队列，按FIFO进出任务)，LinkedBlockingQueue(基于链表的阻塞队列，FIFO方式)，SynchronousQueue(不存储元素的阻塞队列。每个插入操作必须等到另一个线程调用移除操作，否则插入操作一直处于阻塞状态，吞吐量要高于LinkedBlockingQueue，是一个无限阻塞队列)，PriorityBlockingQueue(具有优先级的无限阻塞队列)
3. maximunPoolSize：线程池最大数量，也就是线程池允许创建的最大线程数。这个参数跟阻塞队列的关系是这样的：如果阻塞队列满了，这个时候又来了一个任务，那么这个时候如果当前线程数小于最大数量，那么就会直接创建新的线程执行任务。当然，如果工作队列是无界的，那么这个参数就没有意义了，因为队列无界，任务都会在队列中存储着
4. ThreadFactory：创建线程的工厂
5. RejectedExecutionHandler：饱和策略，也就是当队列和线程数目都满了以后，采取的策略。有AbortPolicy(直接抛出异常)，CallerRunsPolicy(只用调用者所在线程来运行任务)，DiscardOldestPolicy(丢弃队列里最近的一个任务，并执行当前任务)，DiscardPolicy(不处理，直接丢弃)。当然，还可以自定义策略
6. keeyAliveTime：存活时间。如果当前线程池中的线程数量比基本数量要多，并且是闲置状态的话，这些闲置的线程能存活的最大时间
7. TimeUnit，跟第6个参数一起使用，表示存活时间的时间单位

这些参数中，有3个参数跟线程池大小有关，分别是基本数量，最大数量和阻塞队列。

线程池的处理流程是这样的：

1. 首先判断线程池的基本大小，如果基本大小还没满，那么直接创建新的线程执行任务，否则进行下一步
2. 判断线程池中的阻塞队列是是否已满，没满的话存到阻塞队列里等待执行，否则执行下一步(所以如果是个无界的阻塞队列，那么这一步永远都成立)
3. 判断线程池最大大小是否已满，没满的话直接创建线程执行任务，否则交给饱和策略处理


以1个例子来说明：

    ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(1, 20, 10, TimeUnit.SECONDS,new LinkedBlockingQueue(10));

    for(int i = 0; i < 15; i ++) {
        threadPoolExecutor.submit(new MyThread(i + 1));
    }
    
    static class MyThread implements Runnable {
        private int index;
        public MyThread(int index) {
            this.index = index;
        }
        @Override
        public void run() {
            System.out.println(this.index);
            try {
                Thread.sleep(3000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
   
输出：
    
    1
	12
	13
	14
	15
	2
	3
	4
	5
	6
	7
	8
	9
	10
	11

很明显地看到，线程池基本大小只有1个，所以先输出10，接下来阻塞队列有最多有10个线程等待需要执行，但是多出了12，13，14，15这4个线程，由于最大线程数是20个，所以这4个线程就直接被创建开始执行了。


下面就分析Executors的静态方法：

newFixedThreadPool方法:

	public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }

创建一个基本大小和最大大小都为nThreads，阻塞队列长度为Integer.MAX_VALUE的线程池。

newSingleThreadExecutor方法：

	public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }
    
创建一个基本大小和最大大小都为1，阻塞队列长度为Integer.MAX_VALUE的线程池。

newCachedThreadPool方法：

	public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }
    
创建一个基本大小为0，最大大小都为Integer.MAX_VALUE，阻塞队列为无界队列的线程池。

newScheduledThreadPool方法：

	public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
        return new ScheduledThreadPoolExecutor(corePoolSize);
    }
    
创建一个基本大小为corePoolSize，最大大小都为Integer.MAX_VALUE，阻塞队列为DelayedWorkQueue的线程池。

newSingleThreadScheduledExecutor方法：


创建一个基本大小为corePoolSize，最大大小都为Integer.MAX_VALUE，阻塞队列为DelayedWorkQueue的线程池。

	public static ScheduledExecutorService newSingleThreadScheduledExecutor() {
		return new DelegatedScheduledExecutorService
            (new ScheduledThreadPoolExecutor(1));    
	}

创建一个基本大小为1，最大大小都为Integer.MAX_VALUE，阻塞队列为DelayedWorkQueue的线程池。