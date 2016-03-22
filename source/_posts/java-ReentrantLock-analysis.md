title: Java可重入锁ReentrantLock分析
date: 2016-03-19 14:31:58
tags:
- java
- thread
categories:
- java

--------------

![](http://7x2wh6.com1.z0.glb.clouddn.com/ReentrantLock.png)
Java中的可重入锁ReentrantLock很常见，可以用它来代替内置锁synchronized，ReentrantLock是语法级别的锁，所以比内置锁更加灵活。

<!--more-->


下面这段代码是ReentrantLock的一个例子：


	class Context {
        private ReentrantLock lock = new ReentrantLock();
        public void method() {
            lock.lock();
            System.out.println("do atomic operation");
            try {
                Thread.sleep(3000l);
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                lock.unlock();
            }
        }
    }

    class MyThread implements Runnable {
        private Context context;
        public MyThread(Context context) {
            this.context = context;
        }
        @Override
        public void run() {
            context.method();
        }
    }
    
main方法：

	public static void main(String[] args) {
        Context context = new Context();
        ExecutorService executorService = Executors.newFixedThreadPool(5);
        for(int i = 0; i < 5; i ++ ) {
            executorService.submit(new MyThread(context));
        }
    }
    
    
输出结果，每隔3秒输出：

	do atomic operation   
	
如果没有使用可重入锁的话，那么一次性输出5条 do atomic operation。

ReentrantLock中有3个内部类，分别是Sync、FairSync和NonfairSync。

Sync是一个继承AQS的抽象类，使用独占锁，复写了tryRelease方法。tryAcquire方法由它的两个FairSync(公平锁)和NonfairSync(非公平锁)实现。

AQS相关的内容可以参考文章末尾的参考资料，这篇文章写得非常棒。

ReentrantLock的lock方法使用sync的lock方法，Sync的lock方法是个抽象方法，由公平锁和非公平锁去实现。unlock方法直接使用AQS的release方法。所以说公平锁和非公平锁的释放锁过程是一样的，不一样的是获取锁过程。

先来看一下unlock方法，unlock方法调用的AQS的release方法，也就是调用了tryRelease方法，tryRelease方法调完之后恢复第一个挂起的线程：

	protected final boolean tryRelease(int releases) {
        int c = getState() - releases; // 释放
        if (Thread.currentThread() != getExclusiveOwnerThread()) // 如果当前线程不是独占线程，直接抛出异常
            throw new IllegalMonitorStateException();
        boolean free = false;
        if (c == 0) { // 由于是可重入锁，需要判断是否全部释放了
            free = true;
            setExclusiveOwnerThread(null); // 全部释放的话直接把独占线程设置为null
        }
        setState(c);
        return free;
    }
    
    // 恢复线程
    public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;  // 恢复第一个挂起的线程
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }

ReentrantLock的lock方法就是获取锁的方法，AQS中线程对锁的的竞争结果只有两种，要么获取到了锁；要么没有获取到锁，没有获取的锁线程被挂起等待被唤醒。
    
公平锁FairSync的lock方法：

	final void lock() {
		// acquire方法内部调用tryAcquire方法
		// 公平锁的获取锁方法，对于没有获取到的线程，会按照队列的方式挂起线程
        acquire(1);
    }

	protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
        	// 公平锁这里多了一个!hasQueuedPredecessors()判断，表示是否有线程在队列里等待的时间比当前线程要长，如果有等待时间更长的线程，那么放弃获取锁
            if (!hasQueuedPredecessors() &&
                compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }

非公平锁NonfairSync的lock方法：

	final void lock() {
		// 非公平锁的获取锁
		// 跟公平锁的区别就在这里。直接对状态位state进行cas操作，成功就获取锁，这是一种抢占式的方式。不成功跟公平锁一样进入队列挂起线程
        if (compareAndSetState(0, 1))
            setExclusiveOwnerThread(Thread.currentThread());
        else
            acquire(1);
    }

	// 调用Sync的nonfairTryAcquire方法
	protected final boolean tryAcquire(int acquires) {
        return nonfairTryAcquire(acquires);
    }

	final boolean nonfairTryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            if (compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0) // overflow
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
    

如上述源码和注释所说，公平锁和非公平锁的最主要区别就是获取锁的方式不一样。

公平锁获取锁的时候，首先先读取状态位state，然后再做判断，之后使用cas设置状态位。能获取锁的线程就获取锁，不能获取锁的线程被挂起进入队列。之后再来的线程的等待时间没有已经在队列里的线程等待时间长，所以会一直进入等待队列。 公平锁类似于排队买火车票一样，后面来的人没有前面来的人等待时间长，会一直在队尾被加入到队列里。

非公平锁获取锁的时候，立马就使用cas判断设置状态位，是一种抢占式的方式。同时非公平锁也没有等待时间长的线程会优先获取锁这个概念。非公平锁类似吃饭排队，但是总会有那么几个人试图插队。

公平锁和非公平锁的还有另外一个差别，前面已经分析过了。就是公平锁获取锁多了一个判断条件，当前线程的等待时间没有队列里的线程等待时间长的话，不能获取锁；而非公平锁没有这个条件。


ReentrantLock的默认构造函数使用的是NonfairSync，如果想使用FairSync，使用带有boolean参数的构造函数，传入true表示FairSync，否则是NonfairSync。


ReentrantLock内部还提供了一些有用的方法：

hasQueuedThreads： 查询是否有线程在等待队列里
hasQueuedThread(Thread thread)：查询线程是否在等待队列里
isHeldByCurrentThread：当前线程是否持有锁
getQueueLength：队列中的挂起线程个数

等等还有其他的一些有用方法。

总结：

ReentrantLock可重入锁内部有3个类，Sync、FairSync和NonfairSync。

Sync是一个继承AQS的抽象类，并发的控制就是通过Sync实现的(当然是使用AQS实现的，AQS是Java并发包的一个同步基础类)，它复写了tryRelease方法，它有2个子类FairSync和NonfairSync，也就是公平锁和非公平锁。

由于Sync复写了tryRelease方法，它的2个子类公平锁和非公平锁没有再次复写这个方法，所以公平锁和非公平锁的释放锁操作是一样的，释放锁也就是唤醒等待队列中的第一个被挂起的线程。

虽然公平锁和非公平锁的释放锁方式一样，但是它们的获取锁方式不一样，公平锁获取锁的时候，如果1个线程获取到了锁，其他线程都会被挂起并且进入等待队列，后面来的线程的等待时间没有队列里的线程等待时间长的话，那么就放弃获取锁，进入等待队列。非公平锁获取锁的方式是一种抢占式的方式，不考虑等待时间的问题，无论哪个线程获取到了锁，其他线程就进入等待队列。


参考资料：

[http://ifeve.com/jdk1-8-abstractqueuedsynchronizer/](http://ifeve.com/jdk1-8-abstractqueuedsynchronizer/)

