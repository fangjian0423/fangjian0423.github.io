title: Java实现同步的几种方式
date: 2016-04-18 20:57:57
tags:
- java
categories: java

----------------

Java提供了很多同步操作，比如synchronized关键字、wait/notifyAll、ReentrantLock、Condition、一些并发包下的工具类、Semaphore，ThreadLocal、AbstractQueuedSynchronizer等。

本文简单说明一下这几种方式的使用。

![](http://7x2wh6.com1.z0.glb.clouddn.com/synchronize-way.jpeg)

<!--more-->

## ReentrantLock可重入锁 ##


ReentrantLock可重入锁是jdk内置的一个锁对象，可以用来实现同步，基本使用方法如下：

	public class ReentrantLockTest {
    
        private ReentrantLock lock = new ReentrantLock();
    
        public void execute() {
            lock.lock();
            try {
                System.out.println(Thread.currentThread().getName() + " do something synchronize");
                try {
                    Thread.sleep(5000l);
                } catch (InterruptedException e) {
                    System.err.println(Thread.currentThread().getName() + " interrupted");
                    Thread.currentThread().interrupt();
                }
            } finally {
                lock.unlock();
            }
        }
    
        public static void main(String[] args) {
            ReentrantLockTest reentrantLockTest = new ReentrantLockTest();
            Thread thread1 = new Thread(new Runnable() {
                @Override
                public void run() {
                    reentrantLockTest.execute();
                }
            });
            Thread thread2 = new Thread(new Runnable() {
                @Override
                public void run() {
                    reentrantLockTest.execute();
                }
            });
            thread1.start();
            thread2.start();
        }
    
    }
    
输出：
    
	Thread-0 do something synchronize
	// 隔了5秒钟 输入下面
	Thread-1 do something synchronize

这个例子表示同一时间段只能有1个线程执行execute方法。

可重入锁中可重入表示的意义在于**对于同一个线程，可以继续调用加锁的方法，而不会被挂起**。可重入锁内部维护一个计数器，对于同一个线程调用lock方法，计数器+1，调用unlock方法，计数器-1。

举个例子再次说明一下可重入的意思，在一个加锁方法execute中调用另外一个加锁方法anotherLock并不会被挂起，可以直接调用(调用execute方法时计数器+1，然后内部又调用了anotherLock方法，计数器+1，变成了2)：


	public void execute() {
        lock.lock();
        try {
            System.out.println(Thread.currentThread().getName() + " do something synchronize");
            try {
                anotherLock();
                Thread.sleep(5000l);
            } catch (InterruptedException e) {
                System.err.println(Thread.currentThread().getName() + " interrupted");
                Thread.currentThread().interrupt();
            }
        } finally {
            lock.unlock();
        }
    }

    public void anotherLock() {
        lock.lock();
        try {
            System.out.println(Thread.currentThread().getName() + " invoke anotherLock");
        } finally {
            lock.unlock();
        }
    }
    
输出：

	Thread-0 do something synchronize
	Thread-0 invoke anotherLock
	// 隔了5秒钟 输入下面
	Thread-1 do something synchronize
	Thread-1 invoke anotherLock
	
	
## synchronized关键字 ##

synchronized关键跟ReentrantLock一样，也支持可重入锁。但是它是一个关键字，是一种语法级别的同步方式，称为内置锁：

	public class SynchronizedKeyWordTest {
    
        public synchronized void execute() {
                System.out.println(Thread.currentThread().getName() + " do something synchronize");
            try {
                anotherLock();
                Thread.sleep(5000l);
            } catch (InterruptedException e) {
                System.err.println(Thread.currentThread().getName() + " interrupted");
                Thread.currentThread().interrupt();
            }
        }
    
        public synchronized void anotherLock() {
            System.out.println(Thread.currentThread().getName() + " invoke anotherLock");
        }
    
        public static void main(String[] args) {
            SynchronizedKeyWordTest reentrantLockTest = new SynchronizedKeyWordTest();
            Thread thread1 = new Thread(new Runnable() {
                @Override
                public void run() {
                    reentrantLockTest.execute();
                }
            });
            Thread thread2 = new Thread(new Runnable() {
                @Override
                public void run() {
                    reentrantLockTest.execute();
                }
            });
            thread1.start();
            thread2.start();
        }
    
    }

输出结果跟ReentrantLock一样，这个例子说明内置锁可以作用在方法上。它还可以作用到变量，静态方法上。

synchronized跟ReentrantLock相比，有几点局限性：

1. 加锁的时候不能设置超时。ReentrantLock有提供tryLock方法，可以设置超时时间，如果超过了这个时间并且没有获取到锁，就会放弃，而synchronized却没有这种功能
2. ReentrantLock可以使用多个Condition，而synchronized却只能有1个
3. 不能中断一个试图获得锁的线程
4. ReentrantLock可以选择公平锁和非公平锁
5. ReentrantLock可以获得正在等待线程的个数，计数器等


## Condition条件对象 ##

条件对象的意义在于对于一个已经获取锁的线程，如果还需要等待其他条件才能继续执行的情况下，才会使用Condition条件对象。

	public class ConditionTest {
    
        public static void main(String[] args) {
            ReentrantLock lock = new ReentrantLock();
            Condition condition = lock.newCondition();
            Thread thread1 = new Thread(new Runnable() {
                @Override
                public void run() {
                    lock.lock();
                    try {
                        System.out.println(Thread.currentThread().getName() + " run");
                        System.out.println(Thread.currentThread().getName() + " wait for condition");
                        try {
                            condition.await();
                            System.out.println(Thread.currentThread().getName() + " continue");
                        } catch (InterruptedException e) {
                            System.err.println(Thread.currentThread().getName() + " interrupted");
                            Thread.currentThread().interrupt();
                        }
                    } finally {
                        lock.unlock();
                    }
                }
            });
            Thread thread2 = new Thread(new Runnable() {
                @Override
                public void run() {
                    lock.lock();
                    try {
                        System.out.println(Thread.currentThread().getName() + " run");
                        System.out.println(Thread.currentThread().getName() + " sleep 5 secs");
                        try {
                            Thread.sleep(5000l);
                        } catch (InterruptedException e) {
                            System.err.println(Thread.currentThread().getName() + " interrupted");
                            Thread.currentThread().interrupt();
                        }
                        condition.signalAll();
                    } finally {
                        lock.unlock();
                    }
                }
            });
            thread1.start();
            thread2.start();
        }
    
    }

这个例子中thread1执行到condition.await()时，当前线程会被挂起，直到thread2调用了condition.signalAll()方法之后，thread1才会重新被激活执行。

这里需要注意的是thread1调用Condition的await方法之后，thread1线程释放锁，然后马上加入到Condition的等待队列，由于thread1释放了锁，thread2获得锁并执行，thread2执行signalAll方法之后，Condition中的等待队列thread1被取出并加入到AQS中，接下来thread2执行完毕之后释放锁，由于thread1已经在AQS的等待队列中，所以thread1被唤醒，继续执行。

## wait/notifyAll 方式 ##

wait/notifyAll方式跟ReentrantLock/Condition方式的原理是一样的。

Java中每个对象都拥有一个内置锁，在内置锁中调用wait，notify方法相当于调用锁的Condition条件对象的await和signalAll方法。

使用wait/notifyAll实现上面的那个Condition例子：

	public class WaitNotifyAllTest {
    
        public synchronized void doWait() {
            System.out.println(Thread.currentThread().getName() + " run");
            System.out.println(Thread.currentThread().getName() + " wait for condition");
            try {
                this.wait();
                System.out.println(Thread.currentThread().getName() + " continue");
            } catch (InterruptedException e) {
                System.err.println(Thread.currentThread().getName() + " interrupted");
                Thread.currentThread().interrupt();
            }
        }
    
        public synchronized void doNotify() {
            try {
                System.out.println(Thread.currentThread().getName() + " run");
                System.out.println(Thread.currentThread().getName() + " sleep 5 secs");
                Thread.sleep(5000l);
                this.notifyAll();
            } catch (InterruptedException e) {
                System.err.println(Thread.currentThread().getName() + " interrupted");
                Thread.currentThread().interrupt();
            }
        }
    
        public static void main(String[] args) {
            WaitNotifyAllTest waitNotifyAllTest = new WaitNotifyAllTest();
            Thread thread1 = new Thread(new Runnable() {
                @Override
                public void run() {
                    waitNotifyAllTest.doWait();
                }
            });
            Thread thread2 = new Thread(new Runnable() {
                @Override
                public void run() {
                    waitNotifyAllTest.doNotify();
                }
            });
            thread1.start();
            thread2.start();
        }
    
    }
    
这里需要注意的是由于Condition是由锁创建的，所以调用wait/notifyAll方法的时候需要获得当前线程的锁，否则会发生IllegalMonitorStateException异常。


## ThreadLocal ##

ThreadLocal是一种把变量放到线程本地的方式来实现线程同步的。

比如SimpleDateFormat不是一个线程安全的类，可以使用ThreadLocal实现同步。

	public class ThreadLocalTest {
    
        private static ThreadLocal<SimpleDateFormat> dateFormatThreadLocal = new ThreadLocal<SimpleDateFormat>() {
            @Override
            protected SimpleDateFormat initialValue() {
                return new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
            }
        };
    
        public static void main(String[] args) {
            Thread thread1 = new Thread(new Runnable() {
                @Override
                public void run() {
                    Date date = new Date();
                    System.out.println(dateFormatThreadLocal.get().format(date));
                }
            });
            Thread thread2 = new Thread(new Runnable() {
                @Override
                public void run() {
                    Date date = new Date();
                    System.out.println(dateFormatThreadLocal.get().format(date));
                }
            });
            thread1.start();
            thread2.start();
        }
    
    }

## Semaphore信号量 ##

Semaphore信号量被用于控制特定资源在同一个时间被访问的个数。类似连接池的概念，保证资源可以被合理的使用。可以使用构造器初始化资源个数：

	public class SemaphoreTest {
    
        private static Semaphore semaphore = new Semaphore(2);
    
        public static void main(String[] args) {
            for(int i = 0; i < 5; i ++) {
                new Thread(new Runnable() {
                    @Override
                    public void run() {
                        try {
                            semaphore.acquire();
                            System.out.println(Thread.currentThread().getName() + " " + new Date());
                            Thread.sleep(5000l);
                            semaphore.release();
                        } catch (InterruptedException e) {
                            System.err.println(Thread.currentThread().getName() + " interrupted");
                        }
                    }
                }).start();
            }
        }
    
    }

输出：

	Thread-1 Mon Apr 18 18:03:46 CST 2016
	Thread-0 Mon Apr 18 18:03:46 CST 2016
	Thread-3 Mon Apr 18 18:03:51 CST 2016
	Thread-2 Mon Apr 18 18:03:51 CST 2016
	Thread-4 Mon Apr 18 18:03:56 CST 2016
	
## 并发包下的工具类 ##

一般情况下，我们不会使用wait/notifyAll或者ReentrantLock这种比较底层的类，而是使用并发包下提供的一些工具类。

### CountDownLatch ###

CountDownLatch是一个计数器，它的构造方法中需要设置一个数值，用来设定计数的次数。每次调用countDown()方法之后，这个计数器都会减去1，CountDownLatch会一直阻塞着调用await()方法的线程，直到计数器的值变为0。

	public class CountDownLatchTest {
    
        public static void main(String[] args) {
            CountDownLatch countDownLatch = new CountDownLatch(5);
            for(int i = 0; i < 5; i ++) {
                new Thread(new Runnable() {
                    @Override
                    public void run() {
                        System.out.println(Thread.currentThread().getName() + " " + new Date() + " run");
                        try {
                            Thread.sleep(5000l);
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                        countDownLatch.countDown();
                    }
                }).start();
            }
            try {
                countDownLatch.await();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("all thread over");
        }
    
    }
    
输出：

	Thread-2 Mon Apr 18 18:18:30 CST 2016 run
	Thread-3 Mon Apr 18 18:18:30 CST 2016 run
	Thread-4 Mon Apr 18 18:18:30 CST 2016 run
	Thread-0 Mon Apr 18 18:18:30 CST 2016 run
	Thread-1 Mon Apr 18 18:18:30 CST 2016 run
	all thread over

### CyclicBarrier ###

CyclicBarrier阻塞调用的线程，直到条件满足时，阻塞的线程同时被打开。

调用await()方法的时候，这个线程就会被阻塞，当调用await()的线程数量到达屏障数的时候，主线程就会取消所有被阻塞线程的状态。

在CyclicBarrier的构造方法中，还可以设置一个barrierAction。

在所有的屏障都到达之后，会启动一个线程来运行这里面的代码。

	public class CyclicBarrierTest {
    
        public static void main(String[] args) {
            Random random = new Random();
            CyclicBarrier cyclicBarrier = new CyclicBarrier(5);
            for(int i = 0; i < 5; i ++) {
                new Thread(new Runnable() {
                    @Override
                    public void run() {
                        int secs = random.nextInt(5);
                        System.out.println(Thread.currentThread().getName() + " " + new Date() + " run, sleep " + secs + " secs");
                        try {
                            Thread.sleep(secs * 1000);
                            cyclicBarrier.await();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        } catch (BrokenBarrierException e) {
                            e.printStackTrace();
                        }
                        System.out.println(Thread.currentThread().getName() + " " + new Date() + " runs over");
                    }
                }).start();
            }
        }
    
    }

相比CountDownLatch，CyclicBarrier是可以被循环使用的，而且遇到线程中断等情况时，还可以利用reset()方法，重置计数器，从这些方面来说，CyclicBarrier会比CountDownLatch更加灵活一些。		

		
## AbstractQueuedSynchronizer ##

AQS是很多同步工具类的基础，比如ReentrentLock里的公平锁和非公平锁，Semaphore里的公平锁和非公平锁，CountDownLatch里的锁等他们的底层都是使用AbstractQueuedSynchronizer完成的。

基于AbstractQueuedSynchronizer自定义实现一个独占锁：

	public class MySynchronizer extends AbstractQueuedSynchronizer {
    
        @Override
        protected boolean tryAcquire(int arg) {
            if(compareAndSetState(0, 1)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }
    
        @Override
        protected boolean tryRelease(int arg) {
            setState(0);
            setExclusiveOwnerThread(null);
            return true;
        }
    
        public void lock() {
            acquire(1);
        }
    
        public void unlock() {
            release(1);
        }
    
        public static void main(String[] args) {
            MySynchronizer mySynchronizer = new MySynchronizer();
            Thread thread1 = new Thread(new Runnable() {
                @Override
                public void run() {
                    mySynchronizer.lock();
                    try {
                        System.out.println(Thread.currentThread().getName() + " run");
                        System.out.println(Thread.currentThread().getName() + " will sleep 5 secs");
                        try {
                            Thread.sleep(5000l);
                            System.out.println(Thread.currentThread().getName() + " continue");
                        } catch (InterruptedException e) {
                            System.err.println(Thread.currentThread().getName() + " interrupted");
                            Thread.currentThread().interrupt();
                        }
                    } finally {
                        mySynchronizer.unlock();
                    }
                }
            });
            Thread thread2 = new Thread(new Runnable() {
                @Override
                public void run() {
                    mySynchronizer.lock();
                    try {
                        System.out.println(Thread.currentThread().getName() + " run");
                    } finally {
                        mySynchronizer.unlock();
                    }
                }
            });
            thread1.start();
            thread2.start();
        }
    
    }

MySynchronizer并没有实现可重入功能，只是简单的一个独占锁。
