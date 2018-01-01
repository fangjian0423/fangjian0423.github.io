title: CountDownLatch和CyclicBarrier的区别
date: 2016-05-01 01:59:39
tags:
- java
categories: java

----------------

CountDownLatch和CyclicBarrier都是java并发包下的工具类。

CountDownLatch用于处理一个或多个线程等待其他所有线程完毕之后再继续进行操作。

比如要处理一个非常耗时的任务，处理完之后需要更新这个任务的状态，需要开多线程去分批次处理任务中的各个子任务，当所有的子任务全部执行完毕之后，就可以更新任务状态了。这个时候就需要使用CountDownLatch。

CyclicBarrier用于N个线程相互等待，当到达条件之后所有线程继续执行。

比如一个抽奖活动，每个线程进行抽奖，当奖品全部抽完之后对各个线程中的用户进行后续操作。

个人理解的两者之间的区别有3点：

1. CountDownLatch可以阻塞1个或N个线程，CyclicBarrier必须要阻塞N个线程
2. CountDownLatch用完之后就不能再次使用了，CyclicBarrier用完之后可以再次使用，CyclicBarrier还可以做reset操作
3. CountDownLatch底层使用的是共享锁，CyclicBarrier底层使用的是ReentrantLock和这个lock的条件对象Condition

<!--more-->

## 例子 ##

1个CountDownLatch例子，这个例子阻塞3个线程，分别是主线程，Thread1和Thread2。这3个线程会在调用await方法之后阻塞，直到计数器变成0：

	public class CountDownLatchTest {
    
        public static void main(String[] args) {
            System.out.println("主任务开始，一共需要进行7个子任务。第1和第2个子任务需要进行后续操作 " + DateFormatUtils.format(new Date(), "yyyy-MM-dd HH:mm:ss"));
            CountDownLatch countDownLatch = new CountDownLatch(5);
            for(int i = 0; i < 7; i ++) {
                final int index = i;
                new Thread(new Runnable() {
                    @Override
                    public void run() {
                        System.out.println("子任务在线程 " + Thread.currentThread().getName() + " 中运行 " + DateFormatUtils.format(new Date(), "yyyy-MM-dd HH:mm:ss"));
                        if(index == 1 || index == 2) {
                            try {
                                countDownLatch.await();
                            } catch (InterruptedException e) {
                                e.printStackTrace();
                            }
                            System.out.println("子任务在线程 " + Thread.currentThread().getName() + " 中进行后续操作 " + DateFormatUtils.format(new Date(), "yyyy-MM-dd HH:mm:ss"));
                        }
                        if(index != 1 && index != 2) {
                            try {
                                Thread.sleep(5000l);
                            } catch (InterruptedException e) {
                                e.printStackTrace();
                            }
                            countDownLatch.countDown();
                        }
                    }
                }, "Thread-" + i).start();
            }
            try {
                countDownLatch.await();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("主任务结束 " + DateFormatUtils.format(new Date(), "yyyy-MM-dd HH:mm:ss"));
        }
    
    }
    
输出：

	主任务开始，一共需要进行7个子任务。第1和第2个子任务需要进行后续操作 2016-05-01 00:41:57
	子任务在线程 Thread-1 中运行 2016-05-01 00:41:57
	子任务在线程 Thread-2 中运行 2016-05-01 00:41:57
	子任务在线程 Thread-0 中运行 2016-05-01 00:41:57
	子任务在线程 Thread-3 中运行 2016-05-01 00:41:57
	子任务在线程 Thread-4 中运行 2016-05-01 00:41:57
	子任务在线程 Thread-5 中运行 2016-05-01 00:41:57
	子任务在线程 Thread-6 中运行 2016-05-01 00:41:57
	子任务在线程 Thread-2 中进行后续操作 2016-05-01 00:42:02
	主任务结束 2016-05-01 00:42:02
	子任务在线程 Thread-1 中进行后续操作 2016-05-01 00:42:02

1个CyclicBarrier例子，模拟抽奖，每个用户都可以抽奖，当所有的用户抽完奖之后才能开始颁发奖项：


	public class CyclicBarrierTest {
    
        public static void main(String[] args) throws InterruptedException {
            System.out.println("5个用户开始抽奖" + DateFormatUtils.format(new Date(), "yyyy-MM-dd HH:mm:ss"));
            CyclicBarrier cyclicBarrier = new CyclicBarrier(5);
            for(int i = 0; i < 5; i ++) {
                final int index = i;
                new Thread(new Runnable() {
                    @Override
                    public void run() {
                        System.out.println(Thread.currentThread().getName() + " 用户开始抽奖，持续"+(index+1)+"秒" + DateFormatUtils.format(new Date(), "yyyy-MM-dd HH:mm:ss"));
                        try {
                            Thread.sleep((index + 1) * 1000);
                            cyclicBarrier.await();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        } catch (BrokenBarrierException e) {
                            e.printStackTrace();
                        }
                        System.out.println("所有用户抽奖完毕，颁发奖项。为用户" + Thread.currentThread().getName() + "颁奖。" + DateFormatUtils.format(new Date(), "yyyy-MM-dd HH:mm:ss"));
                    }
                }, "Thread-" + i).start();
            }
        }
    
    }

输出：

	5个用户开始抽奖2016-05-01 00:57:16
	Thread-1 用户开始抽奖，持续2秒2016-05-01 00:57:16
	Thread-2 用户开始抽奖，持续3秒2016-05-01 00:57:16
	Thread-3 用户开始抽奖，持续4秒2016-05-01 00:57:16
	Thread-4 用户开始抽奖，持续5秒2016-05-01 00:57:16
	Thread-0 用户开始抽奖，持续1秒2016-05-01 00:57:16
	所有用户抽奖完毕，颁发奖项。为用户Thread-0颁奖。2016-05-01 00:57:21
	所有用户抽奖完毕，颁发奖项。为用户Thread-2颁奖。2016-05-01 00:57:21
	所有用户抽奖完毕，颁发奖项。为用户Thread-1颁奖。2016-05-01 00:57:21
	所有用户抽奖完毕，颁发奖项。为用户Thread-4颁奖。2016-05-01 00:57:21
	所有用户抽奖完毕，颁发奖项。为用户Thread-3颁奖。2016-05-01 00:57:21

CyclicBarrier中的计数器到0之后，可以重用：

	public class CyclicBarrierTest2 {
    
        public static void main(String[] args) throws InterruptedException {
            System.out.println("5个用户开始抽奖" + DateFormatUtils.format(new Date(), "yyyy-MM-dd HH:mm:ss"));
            CyclicBarrier cyclicBarrier = new CyclicBarrier(5);
            for(int i = 0; i < 5; i ++) {
                final int index = i;
                new Thread(new Runnable() {
                    @Override
                    public void run() {
                        System.out.println(Thread.currentThread().getName() + " 用户开始抽奖，持续"+(index+1)+"秒" + DateFormatUtils.format(new Date(), "yyyy-MM-dd HH:mm:ss"));
                        try {
                            Thread.sleep((index + 1) * 1000);
                            cyclicBarrier.await();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        } catch (BrokenBarrierException e) {
                            e.printStackTrace();
                        }
                        System.out.println("所有用户抽奖完毕，颁发奖项。为用户" + Thread.currentThread().getName() + "颁奖。" + DateFormatUtils.format(new Date(), "yyyy-MM-dd HH:mm:ss"));
                    }
                }, "Thread-" + i).start();
            }
            Thread.sleep(5000l);
            System.out.println("下一轮抽奖开始" + DateFormatUtils.format(new Date(), "yyyy-MM-dd HH:mm:ss"));
            for(int i = 0; i < 5; i ++) {
                final int index = i;
                new Thread(new Runnable() {
                    @Override
                    public void run() {
                        System.out.println(Thread.currentThread().getName() + " 用户开始抽奖，持续"+(index+1)+"秒" + DateFormatUtils.format(new Date(), "yyyy-MM-dd HH:mm:ss"));
                        try {
                            Thread.sleep((index + 1) * 1000);
                            cyclicBarrier.await();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        } catch (BrokenBarrierException e) {
                            e.printStackTrace();
                        }
                        System.out.println("所有用户抽奖完毕，颁发奖项。为用户" + Thread.currentThread().getName() + "颁奖。" + DateFormatUtils.format(new Date(), "yyyy-MM-dd HH:mm:ss"));
                    }
                }, "Thread-" + i).start();
            }
        }
    
    }


输出：

	5个用户开始抽奖2016-05-01 00:59:40
	Thread-1 用户开始抽奖，持续2秒2016-05-01 00:59:40
	Thread-2 用户开始抽奖，持续3秒2016-05-01 00:59:40
	Thread-3 用户开始抽奖，持续4秒2016-05-01 00:59:40
	Thread-4 用户开始抽奖，持续5秒2016-05-01 00:59:40
	Thread-0 用户开始抽奖，持续1秒2016-05-01 00:59:40
	所有用户抽奖完毕，颁发奖项。为用户Thread-4颁奖。2016-05-01 00:59:45
	所有用户抽奖完毕，颁发奖项。为用户Thread-0颁奖。2016-05-01 00:59:45
	所有用户抽奖完毕，颁发奖项。为用户Thread-1颁奖。2016-05-01 00:59:45
	下一轮抽奖开始2016-05-01 00:59:45
	所有用户抽奖完毕，颁发奖项。为用户Thread-3颁奖。2016-05-01 00:59:45
	所有用户抽奖完毕，颁发奖项。为用户Thread-2颁奖。2016-05-01 00:59:45
	Thread-1 用户开始抽奖，持续2秒2016-05-01 00:59:45
	Thread-0 用户开始抽奖，持续1秒2016-05-01 00:59:45
	Thread-2 用户开始抽奖，持续3秒2016-05-01 00:59:45
	Thread-3 用户开始抽奖，持续4秒2016-05-01 00:59:45
	Thread-4 用户开始抽奖，持续5秒2016-05-01 00:59:45
	所有用户抽奖完毕，颁发奖项。为用户Thread-1颁奖。2016-05-01 00:59:50
	所有用户抽奖完毕，颁发奖项。为用户Thread-2颁奖。2016-05-01 00:59:50
	所有用户抽奖完毕，颁发奖项。为用户Thread-0颁奖。2016-05-01 00:59:50
	所有用户抽奖完毕，颁发奖项。为用户Thread-4颁奖。2016-05-01 00:59:50
	所有用户抽奖完毕，颁发奖项。为用户Thread-3颁奖。2016-05-01 00:59:50

## 底层分析 ##


### CountDownLatch ###

CountDownLatch底层使用的是共享锁，它有个内部类Sync，这个Sync继承AQS，实现了共享锁。

简单画了一下共享锁的实现。

比如有4个线程在等待队列里，并且节点类型都是共享锁。 会唤醒head节点的下一节点中的线程Thread1。head节点就变成了之前head节点的下个节点，然后再做重复操作。 这个过程是一个传播过程，会依次唤醒各个共享节点中的线程。

![](http://7x2wh6.com1.z0.glb.clouddn.com/lockqueue01.jpg)

并发包下的另外一个工具类Semaphore底层也是使用共享锁实现的。但是它跟CountDownLatch唯一的区别就是它不会唤醒所有的共享节点中的线程，而是唤醒它能唤醒的最大线程数(由信号量可用大小决定)。

### CyclicBarrier ###

CyclicBarrier底层使用的是ReentrantLock和这个lock的条件对象Condition。

它拥有的属性如下：

	private static class Generation {
        boolean broken = false;
    }

    private final ReentrantLock lock = new ReentrantLock(); // 可重入锁
    private final Condition trip = lock.newCondition(); // 可重入锁的条件对象
    private final int parties; // 计数器原始值，永远不会变
    private final Runnable barrierCommand; // 计数器到了之后需要执行的Runnable，可为空
    private Generation generation = new Generation(); // 一个Generation对象的实例，当计数器为0的时候这个实例将会重新被构造
    private int count; // 计数器当前的值

await方法：

	private int dowait(boolean timed, long nanos)
        throws InterruptedException, BrokenBarrierException,
               TimeoutException {
        final ReentrantLock lock = this.lock;
        lock.lock(); // 加锁，确保每次只有1个线程调用
        try {
            final Generation g = generation;

            if (g.broken) // 查看generation是否已经损坏
                throw new BrokenBarrierException();

            if (Thread.interrupted()) {
                breakBarrier();
                throw new InterruptedException();
            }

            int index = --count; // 计数器减一
            if (index == 0) {  // 如果计数器为0
                boolean ranAction = false;
                try {
                    final Runnable command = barrierCommand;
                    if (command != null) // 如果Runnable不为空，执行run方法。注意，这里是直接调用run方法，而不是启动1个新的线程
                        command.run();
                    ranAction = true;
                    nextGeneration(); // 一个过程结束，重新开始
                    return 0;
                } finally {
                    if (!ranAction)
                        breakBarrier();
                }
            }

            
            for (;;) {
                try {
                    if (!timed)
                        trip.await(); // 放到Conditon的等待队列里，同时释放锁，让其他线程执行await方法
                    else if (nanos > 0L)
                        nanos = trip.awaitNanos(nanos);
                } catch (InterruptedException ie) {
                    if (g == generation && ! g.broken) {
                        breakBarrier();
                        throw ie;
                    } else {
                        // We're about to finish waiting even if we had not
                        // been interrupted, so this interrupt is deemed to
                        // "belong" to subsequent execution.
                        Thread.currentThread().interrupt();
                    }
                }

                if (g.broken)
                    throw new BrokenBarrierException();

                if (g != generation) // 说明执行了nextGeneration方法，计数器到了0
                    return index;

                if (timed && nanos <= 0L) {
                    breakBarrier();
                    throw new TimeoutException();
                }
            }
        } finally {
            lock.unlock(); // 解锁
        }
    }
    
    private void nextGeneration() {
    	// 唤醒Conditon等待队列上的所有线程
        trip.signalAll();
		// 计数器值变成原始值，重新开始
        count = parties;
        // generation被重新构造
        generation = new Generation();
    }
    
执行过程解释：

比如Thread1执行了await方法，这个时候await方法加锁，确保其他线程不能再次调用await方法。

然后在await方法中把计数器数字减一。

如果计数器还没到0：将Thread1加入到Condition的条件队列，同时释放锁。这个时候其他线程就可以获得await方法的锁并执行。

如果计数器到了0：调用Conditon的signalAll方法，把Condition等待队列上的所有线程移除，移到AQS的等待队列里，然后返回index，释放锁，之后AQS等待队列上的节点中的线程就可以被唤醒了。


