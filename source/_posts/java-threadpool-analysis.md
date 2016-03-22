title: Java线程池ThreadPoolExecutor源码分析
date: 2016-03-22 20:56:11
tags:
- java
- thread
categories:
- java

--------------


ThreadPoolExecutor是jdk内置线程池的一个实现，基本上大部分情况都会使用这个线程池完成各项操作。

![](http://7x2wh6.com1.z0.glb.clouddn.com/thread-pool.jpeg)

<!--more-->

本文分析ThreadPoolExecutor的实现原理。


## ThreadPoolExecutor的状态和属性 ##

ThreadPoolExecutor的属性在之前的一篇[java内置的线程池笔记](http://fangjian0423.github.io/2015/07/24/java-poolthread/)文章中解释过了，本文不再解释。

ThreadPoolExecutor线程池有5个状态，分别是：

1. RUNNING：可以接受新的任务，也可以处理阻塞队列里的任务
2. SHUTDOWN：不接受新的任务，但是可以处理阻塞队列里的任务
3. STOP：不接受新的任务，不处理阻塞队列里的任务，中断正在处理的任务
4. TIDYING：过渡状态，也就是说所有的任务都执行完了，当前线程池已经没有有效的线程，这个时候线程池的状态将会TIDYING，并且将要调用terminated方法
5. TERMINATED：终止状态。terminated方法调用完成以后的状态


状态之间可以进行转换：

RUNNING -> SHUTDOWN：手动调用shutdown方法，或者ThreadPoolExecutor要被GC回收的时候调用finalize方法，finalize方法内部也会调用shutdown方法

(RUNNING or SHUTDOWN) -> STOP：调用shutdownNow方法

SHUTDOWN -> TIDYING：当队列和线程池都为空的时候

STOP -> TIDYING：当线程池为空的时候

TIDYING -> TERMINATED：terminated方法调用完成之后


ThreadPoolExecutor内部还保存着线程池的有效线程个数。

状态和线程数在ThreadPoolExecutor内部使用一个整型变量保存，没错，一个变量表示两种含义。

为什么一个整型变量既可以保存状态，又可以保存数量？ 分析一下：

首先，我们知道java中1个整型占4个字节，也就是32位，所以1个整型有32位。

所以整型1用二进制表示就是：00000000000000000000000000000001

整型-1用二进制表示就是：11111111111111111111111111111111(这个是补码，不懂的同学可以看下原码，反码，补码的知识)

在ThreadPoolExecutor，整型中32位的前3位用来表示线程池状态，后3位表示线程池中有效的线程数。

	// 前3位表示状态，所有线程数占29位
	private static final int COUNT_BITS = Integer.SIZE - 3;

线程池容量大小为 1 << 29 - 1 = 00011111111111111111111111111111(二进制)，代码如下

	private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

RUNNING状态 -1 << 29 = 11111111111111111111111111111111 << 29 = 11100000000000000000000000000000(前3位为111)：

	private static final int RUNNING    = -1 << COUNT_BITS;
	
SHUTDOWN状态 0 << 29 = 00000000000000000000000000000000 << 29 = 00000000000000000000000000000000(前3位为000)
	
	private static final int SHUTDOWN   =  0 << COUNT_BITS;
	
STOP状态 1 << 29 = 00000000000000000000000000000001 << 29 = 00100000000000000000000000000000(前3位为001)：

	private static final int STOP       =  1 << COUNT_BITS;
	
TIDYING状态 2 << 29 = 00000000000000000000000000000010 << 29 = 01000000000000000000000000000000(前3位为010)：

	private static final int TIDYING    =  2 << COUNT_BITS;

TERMINATED状态 3 << 29 = 00000000000000000000000000000011 << 29 = 01100000000000000000000000000000(前3位为011)：

    private static final int TERMINATED =  3 << COUNT_BITS;	
清楚状态位之后，下面是获得状态和线程数的内部方法：

	// 得到线程数，也就是后29位的数字。 直接跟CAPACITY做一个与操作即可，CAPACITY就是的值就 1 << 29 - 1 = 00011111111111111111111111111111。 与操作的话前面3位肯定为0，相当于直接取后29位的值
	private static int workerCountOf(int c)  { return c & CAPACITY; }
	
	// 得到状态，CAPACITY的非操作得到的二进制位11100000000000000000000000000000，然后做在一个与操作，相当于直接取前3位的的值
	private static int runStateOf(int c)     { return c & ~CAPACITY; }

	// 或操作。相当于更新数量和状态两个操作
	private static int ctlOf(int rs, int wc) { return rs | wc; }
	
	
线程池初始化状态线程数变量：

	// 初始化状态和数量，状态为RUNNING，线程数为0
	private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
	

## ThreadPoolExecutor执行任务 ##

使用ThreadPoolExecutor执行任务的时候，可以使用execute或submit方法，submit方法如下：

	public Future<?> submit(Runnable task) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<Void> ftask = newTaskFor(task, null);
        execute(ftask);
        return ftask;
    }

很明显地看到，submit方法内部使用了execute方法，而且submit方法是有返回值的。在调用execute方法之前，使用FutureTask包装一个Runnable，这个FutureTask就是返回值。

由于submit方法内部调用execute方法，所以execute方法就是执行任务的方法，来看一下execute方法，execute方法内部分3个步骤进行处理。

1. 如果当前正在执行的Worker数量比corePoolSize(基本大小)要小。直接创建一个新的Worker执行任务，会调用addWorker方法
2. 如果当前正在执行的Worker数量大于等于corePoolSize(基本大小)。将任务放到阻塞队列里，如果阻塞队列没满并且状态是RUNNING的话，直接丢到阻塞队列，否则执行第3步。丢到阻塞队列之后，还需要再做一次验证(丢到阻塞队列之后可能另外一个线程关闭了线程池或者刚刚加入到队列的线程死了)。如果这个时候线程池不在RUNNING状态，把刚刚丢入队列的任务remove掉，调用reject方法，否则查看Worker数量，如果Worker数量为0，起一个新的Worker去阻塞队列里拿任务执行
3. 丢到阻塞失败的话，会调用addWorker方法尝试起一个新的Worker去阻塞队列拿任务并执行任务，如果这个新的Worker创建失败，调用reject方法

上面说的Worker可以暂时理解为一个执行任务的线程。

execute方法源码如下，上面提到的3个步骤对应源码中的3个注释：

	public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        int c = ctl.get();
        if (workerCountOf(c) < corePoolSize) {   // 第一个步骤，满足线程池中的线程大小比基本大小要小
            if (addWorker(command, true)) // addWorker方法第二个参数true表示使用基本大小
                return;
            c = ctl.get();
        }
        if (isRunning(c) && workQueue.offer(command)) { // 第二个步骤，线程池的线程大小比基本大小要大，并且线程池还在RUNNING状态，阻塞队列也没满的情况，加到阻塞队列里
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command)) // 虽然满足了第二个步骤，但是这个时候可能突然线程池关闭了，所以再做一层判断
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        else if (!addWorker(command, false)) // 第三个步骤，直接使用线程池最大大小。addWorker方法第二个参数false表示使用最大大小
            reject(command);
    }

addWorker关系着如何起一个线程，再看addWorker方法之前，先看一下ThreadPoolExecutor的一个内部类Worker, Worker是一个AQS的实现类(为何设计成一个AQS在闲置Worker里会说明)，同时也是一个实现Runnable的类，使用独占锁，它的构造函数只接受一个Runnable参数，内部保存着这个Runnable属性，还有一个thread线程属性用于包装这个Runnable(这个thread属性使用ThreadFactory构造，在构造函数内完成thread线程的构造)，另外还有一个completedTasks计数器表示这个Worker完成的任务数。Worker类复写了run方法，使用ThreadPoolExecutor的runWorker方法(在addWorker方法里调用)，直接启动Worker的话，会调用ThreadPoolExecutor的runWork方法。**需要特别注意的是这个Worker是实现了Runnable接口的，thread线程属性使用ThreadFactory构造Thread的时候，构造的Thread中使用的Runnable其实就是Worker。**下面的Worker的源码：


	private final class Worker	
        extends AbstractQueuedSynchronizer
        implements Runnable
    {
        /**
         * This class will never be serialized, but we provide a
         * serialVersionUID to suppress a javac warning.
         */
        private static final long serialVersionUID = 6138294804551838833L;

        /** Thread this worker is running in.  Null if factory fails. */
        final Thread thread;
        /** Initial task to run.  Possibly null. */
        Runnable firstTask;
        /** Per-thread task counter */
        volatile long completedTasks;

        /**
         * Creates with given first task and thread from ThreadFactory.
         * @param firstTask the first task (null if none)
         */
        Worker(Runnable firstTask) {
        	// 使用ThreadFactory构造Thread，这个构造的Thread内部的Runnable就是本身，也就是Worker。所以得到Worker的thread并start的时候，会执行Worker的run方法，也就是执行ThreadPoolExecutor的runWorker方法
        	setState(-1); 把状态位设置成-1，这样任何线程都不能得到Worker的锁，除非调用了unlock方法。这个unlock方法会在runWorker方法中一开始就调用，这是为了确保Worker构造出来之后，没有任何线程能够得到它的锁，除非调用了runWorker之后，其他线程才能获得Worker的锁
            this.firstTask = firstTask;
            this.thread = getThreadFactory().newThread(this);
        }

        /** Delegates main run loop to outer runWorker  */
        public void run() {
            runWorker(this);
        }

        // Lock methods
        //
        // The value 0 represents the unlocked state.
        // The value 1 represents the locked state.

        protected boolean isHeldExclusively() {
            return getState() != 0;
        }

        protected boolean tryAcquire(int unused) {
            if (compareAndSetState(0, 1)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }

        protected boolean tryRelease(int unused) {
            setExclusiveOwnerThread(null);
            setState(0);
            return true;
        }

        public void lock()        { acquire(1); }
        public boolean tryLock()  { return tryAcquire(1); }
        public void unlock()      { release(1); }
        public boolean isLocked() { return isHeldExclusively(); }

        void interruptIfStarted() {
            Thread t;
            if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
                try {
                    t.interrupt();
                } catch (SecurityException ignore) {
                }
            }
        }
    }


接下来看一下addWorker源码：

	// 两个参数，firstTask表示需要跑的任务。boolean类型的core参数为true的话表示使用线程池的基本大小，为false使用线程池最大大小
	// 返回值是boolean类型，true表示新任务被接收了，并且执行了。否则是false
	private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c); // 线程池当前状态

            // 这个判断转换成 rs >= SHUTDOWN && (rs != SHUTDOWN || firstTask != null || workQueue.isEmpty)。 
            // 概括为3个条件：
            // 1. 线程池不在RUNNING状态并且状态是STOP、TIDYING或TERMINATED中的任意一种状态
            
            // 2. 线程池不在RUNNING状态，线程池接受了新的任务 
            
            // 3. 线程池不在RUNNING状态，阻塞队列为空。  满足这3个条件中的任意一个的话，拒绝执行任务
            
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;

            for (;;) {
                int wc = workerCountOf(c); // 线程池线程个数
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize)) // 如果线程池线程数量超过线程池最大容量或者线程数量超过了基本大小(core参数为true，core参数为false的话判断超过最大大小)
                    return false; // 超过直接返回false
                if (compareAndIncrementWorkerCount(c)) // 没有超过各种大小的话，cas操作线程池线程数量+1，cas成功的话跳出循环
                    break retry;
                c = ctl.get();  // 重新检查状态
                if (runStateOf(c) != rs) // 如果状态改变了，重新循环操作
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }
		// 走到这一步说明cas操作成功了，线程池线程数量+1
        boolean workerStarted = false; // 任务是否成功启动标识
        boolean workerAdded = false; // 任务是否添加成功标识
        Worker w = null;
        try {
            final ReentrantLock mainLock = this.mainLock; // 得到线程池的可重入锁
            w = new Worker(firstTask); // 基于任务firstTask构造worker
            final Thread t = w.thread; // 使用Worker的属性thread，这个thread是使用ThreadFactory构造出来的
            if (t != null) { // ThreadFactory构造出的Thread有可能是null，做个判断
                mainLock.lock(); // 锁住，防止并发
                try {
                    // 在锁住之后再重新检测一下状态
                    int c = ctl.get();
                    int rs = runStateOf(c);

                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) { // 如果线程池在RUNNING状态或者线程池在SHUTDOWN状态并且任务是个null
                        if (t.isAlive()) // 判断线程是否还活着，也就是说线程已经启动并且还没死掉
                            throw new IllegalThreadStateException(); // 如果存在已经启动并且还没死的线程，抛出异常
                        workers.add(w); // worker添加到线程池的workers属性中，是个HashSet
                        int s = workers.size(); // 得到目前线程池中的线程个数
                        if (s > largestPoolSize) // 如果线程池中的线程个数超过了线程池中的最大线程数时，更新一下这个最大线程数
                            largestPoolSize = s;
                        workerAdded = true; // 标识一下任务已经添加成功
                    }
                } finally {
                    mainLock.unlock(); // 解锁
                }
                if (workerAdded) { // 如果任务添加成功，运行任务，改变一下任务成功启动标识
                    t.start(); // 启动线程，这里的t是Worker中的thread属性，所以相当于就是调用了Worker的run方法
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted) // 如果任务启动失败，调用addWorkerFailed方法
                addWorkerFailed(w);
        }
        return workerStarted;
    }
    
Worker中的线程start的时候，调用Worker本身run方法，这个run方法之前分析过，调用外部类ThreadPoolExecutor的runWorker方法，直接看runWorker方法：

	final void runWorker(Worker w) {
        Thread wt = Thread.currentThread(); // 得到当前线程
        Runnable task = w.firstTask; // 得到Worker中的任务task，也就是用户传入的task
        w.firstTask = null; // 将Worker中的任务置空
        w.unlock(); // allow interrupts。 
        boolean completedAbruptly = true;
        try {
        	// 如果worker中的任务不为空，继续知否，否则使用getTask获得任务。一直死循环，除非得到的任务为空才退出
            while (task != null || (task = getTask()) != null) {
                w.lock();  // 如果拿到了任务，给自己上锁，表示当前Worker已经要开始执行任务了，已经不是闲置Worker(闲置Worker的解释请看下面的线程池关闭)
                // 在执行任务之前先做一些处理。 1. 如果线程池已经处于STOP状态并且当前线程没有被中断，中断线程 2. 如果线程池还处于RUNNING或SHUTDOWN状态，并且当前线程已经被中断了，重新检查一下线程池状态，如果处于STOP状态并且没有被中断，那么中断线程
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                    beforeExecute(wt, task); // 任务执行前需要做什么，ThreadPoolExecutor是个空实现
                    Throwable thrown = null;
                    try {
                        task.run(); // 真正的开始执行任务，调用的是run方法，而不是start方法。这里run的时候可能会被中断，比如线程池调用了shutdownNow方法
                    } catch (RuntimeException x) { // 任务执行发生的异常全部抛出，不在runWorker中处理
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        afterExecute(task, thrown); // 任务执行结束需要做什么，ThreadPoolExecutor是个空实现
                    }
                } finally {
                    task = null;
                    w.completedTasks++; // 记录执行任务的个数
                    w.unlock(); // 执行完任务之后，解锁，Worker变成闲置Worker
                }
            }
            completedAbruptly = false;
        } finally {
            processWorkerExit(w, completedAbruptly); // 回收Worker方法
        }
    }

我们看一下getTask方法是如何获得任务的：

    // 如果发生了以下四件事中的任意一件，那么Worker需要被回收：
    // 1. Worker个数比线程池最大大小要大
    // 2. 线程池处于STOP状态
    // 3. 线程池处于SHUTDOWN状态并且阻塞队列为空
    // 4. 使用超时时间从阻塞队列里拿数据，并且超时之后没有拿到数据(allowCoreThreadTimeOut || workerCount > corePoolSize)
    private Runnable getTask() {
        boolean timedOut = false; // 如果使用超时时间并且也没有拿到任务的标识

        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // 如果线程池是SHUTDOWN状态并且阻塞队列为空的话，worker数量减一，直接返回null(SHUTDOWN状态还会处理阻塞队列任务，但是阻塞队列为空的话就结束了)，如果线程池是STOP状态的话，worker数量建议，直接返回null(STOP状态不处理阻塞队列任务)[方法一开始注释的2，3两点，返回null，开始Worker回收]
            if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
                decrementWorkerCount();
                return null;
            }

            boolean timed;      // 标记从队列中取任务时是否设置超时时间，如果为true说明这个worker可能需要回收，为false的话这个worker会一直存在，并且阻塞当前线程等待阻塞队列中有数据

            for (;;) {
                int wc = workerCountOf(c); // 得到当前线程池Worker个数
                // allowCoreThreadTimeOut属性默认为false，表示线程池中的核心线程在闲置状态下还保留在池中；如果是true表示核心线程使用keepAliveTime这个参数来作为超时时间
                // 如果worker数量比基本大小要大的话，timed就为true，需要进行回收worker
                timed = allowCoreThreadTimeOut || wc > corePoolSize; 

                if (wc <= maximumPoolSize && ! (timedOut && timed)) // 方法一开始注释的1，4两点，会进行下一步worker数量减一
                    break;
                if (compareAndDecrementWorkerCount(c)) // worker数量减一，返回null，之后会进行Worker回收工作
                    return null;
                c = ctl.get();  // 重新检查线程池状态
                if (runStateOf(c) != rs) // 线程池状态改变的话重新开始外部循环，否则继续内部循环
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }

            try {
            	// 如果需要设置超时时间，使用poll方法，否则使用take方法一直阻塞等待阻塞队列新进数据
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();
                if (r != null)
                    return r;
                timedOut = true;
            } catch (InterruptedException retry) {
                timedOut = false; // 闲置Worker被中断
            }
        }
    }
    
    
如果getTask返回的是null，那说明阻塞队列已经没有任务并且当前调用getTask的Worker需要被回收，那么会调用processWorkerExit方法进行回收：

	private void processWorkerExit(Worker w, boolean completedAbruptly) {
        if (completedAbruptly) // 如果Worker没有正常结束流程调用processWorkerExit方法，worker数量减一。如果是正常结束的话，在getTask方法里worker数量已经减一了
            decrementWorkerCount();

        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock(); // 加锁，防止并发问题
        try {
            completedTaskCount += w.completedTasks; // 记录总的完成任务数
            workers.remove(w); // 线程池的worker集合删除掉需要回收的Worker
        } finally {
            mainLock.unlock(); // 解锁
        }

        tryTerminate(); // 尝试结束线程池

        int c = ctl.get();
        if (runStateLessThan(c, STOP)) {  // 如果线程池还处于RUNNING或者SHUTDOWN状态
            if (!completedAbruptly) { // Worker是正常结束流程的话
                int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
                if (min == 0 && ! workQueue.isEmpty())
                    min = 1;
                if (workerCountOf(c) >= min)
                    return; // 不需要新开一个Worker
            }
            // 新开一个Worker代替原先的Worker
            // 新开一个Worker需要满足以下3个条件中的任意一个：
            // 1. 用户执行的任务发生了异常
            // 2. Worker数量比线程池基本大小要小
            // 3. 阻塞队列不空但是没有任何Worker在工作
            addWorker(null, false);
        }
    }

在回收Worker的时候线程池会尝试结束自己的运行，tryTerminate方法：

	final void tryTerminate() {
        for (;;) {
            int c = ctl.get();
            // 满足3个条件中的任意一个，不终止线程池
            // 1. 线程池还在运行，不能终止
            // 2. 线程池处于TIDYING或TERMINATED状态，说明已经在关闭了，不允许继续处理
            // 3. 线程池处于SHUTDOWN状态并且阻塞队列不为空，这时候还需要处理阻塞队列的任务，不能终止线程池
            if (isRunning(c) ||
                runStateAtLeast(c, TIDYING) ||
                (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty()))
                return;
            // 走到这一步说明线程池已经不在运行，阻塞队列已经没有任务，但是还要回收正在工作的Worker
            if (workerCountOf(c) != 0) {
             	// 由于线程池不运行了，调用了线程池的关闭方法，在解释线程池的关闭原理的时候会说道这个方法
                interruptIdleWorkers(ONLY_ONE); // 中断闲置Worker，直到回收全部的Worker。这里没有那么暴力，只中断一个，中断之后退出方法，中断了Worker之后，Worker会回收，然后还是会调用tryTerminate方法，如果还有闲置线程，那么继续中断
                return;
            }
			// 走到这里说明worker已经全部回收了，并且线程池已经不在运行，阻塞队列已经没有任务。可以准备结束线程池了
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock(); // 加锁，防止并发
            try {
                if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) { // cas操作，将线程池状态改成TIDYING
                    try {
                        terminated(); // 调用terminated方法
                    } finally {
                        ctl.set(ctlOf(TERMINATED, 0)); // terminated方法调用完毕之后，状态变为TERMINATED
                        termination.signalAll();
                    }
                    return;
                }
            } finally {
                mainLock.unlock(); // 解锁
            }
            // else retry on failed CAS
        }
    }


解释了这么多，对线程池的启动并且执行任务做一个总结：


首先，构造线程池的时候，需要一些参数。一些重要的参数解释在 [java内置的线程池笔记](http://fangjian0423.github.io/2015/07/24/java-poolthread/) 文章中的结尾已经说明了一下重要参数的意义。

线程池构造完毕之后，如果用户调用了execute或者submit方法的时候，最后都会使用execute方法执行。

execute方法内部分3种情况处理任务：

1. 如果当前正在执行的Worker数量比corePoolSize(基本大小)要小。直接创建一个新的Worker执行任务，会调用addWorker方法
2. 如果当前正在执行的Worker数量大于等于corePoolSize(基本大小)。将任务放到阻塞队列里，如果阻塞队列没满并且状态是RUNNING的话，直接丢到阻塞队列，否则执行第3步
3. 丢到阻塞失败的话，会调用addWorker方法尝试起一个新的Worker去阻塞队列拿任务并执行任务，如果这个新的Worker创建失败，调用reject方法

线程池中的这个基本大小指的是Worker的数量。一个Worker是一个Runnable的实现类，会被当做一个线程进行启动。Worker内部带有一个Runnable属性firstTask，这个firstTask可以为null，为null的话Worker会去阻塞队列拿任务执行，否则会先执行这个任务，执行完毕之后再去阻塞队列继续拿任务执行。

所以说如果Worker数量超过了基本大小，那么任务都会在阻塞队列里，当Worker执行完了它的第一个任务之后，就会去阻塞队列里拿其他任务继续执行。

Worker在执行的时候会根据一些参数进行调节，比如Worker数量超过了线程池基本大小或者超时时间到了等因素，这个时候Worker会被线程池回收，线程池会尽量保持内部的Worker数量不超过基本大小。

另外Worker执行任务的时候调用的是Runnable的run方法，而不是start方法，调用了start方法就相当于另外再起一个线程了。

Worker在回收的时候会尝试终止线程池。尝试关闭线程池的时候，会检查是否还有Worker在工作，检查线程池的状态，没问题的话会将状态过度到TIDYING状态，之后调用terminated方法，terminated方法调用完成之后将线程池状态更新到TERMINATED。

## ThreadPoolExecutor的关闭 ##
    
线程池的启动过程分析好了之后，接下来看线程池的关闭操作：

shutdown方法，关闭线程池，关闭之后阻塞队列里的任务不受影响，会继续被Worker处理，但是新的任务不会被接受：

	public void shutdown() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock(); // 关闭的时候需要加锁，防止并发
        try {
            checkShutdownAccess(); // 检查关闭线程池的权限
            advanceRunState(SHUTDOWN); // 把线程池状态更新到SHUTDOWN
            interruptIdleWorkers(); // 中断闲置的Worker
            onShutdown(); // 钩子方法，默认不处理。ScheduledThreadPoolExecutor会做一些处理
        } finally {
            mainLock.unlock(); // 解锁
        }
        tryTerminate(); // 尝试结束线程池，上面已经分析过了
    }

interruptIdleWorkers方法，注意，这个方法打断的是闲置Worker，打断闲置Worker之后，getTask方法会返回null，然后Worker会被回收。那什么是闲置Worker呢？

闲置Worker是这样解释的：Worker运行的时候会去阻塞队列拿数据(getTask方法)，拿的时候如果没有设置超时时间，那么会一直阻塞等待阻塞队列进数据，这样的Worker就被称为闲置Worker。由于Worker也是一个AQS，在runWorker方法里会有一对lock和unlock操作，这对lock操作是为了确保Worker不是一个闲置Worker。

所以Worker被设计成一个AQS是为了根据Worker的锁来判断是否是闲置线程，是否可以被强制中断。


下面我们看下interruptIdleWorkers方法：

	// 调用他的一个重载方法，传入了参数false，表示要中断所有的正在运行的闲置Worker，如果为true表示只打断一个闲置Worker
	private void interruptIdleWorkers() {
        interruptIdleWorkers(false);
    }
    
    private void interruptIdleWorkers(boolean onlyOne) {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock(); // 中断闲置Worker需要加锁，防止并发
        try {
            for (Worker w : workers) { 
                Thread t = w.thread; // 拿到worker中的线程
                if (!t.isInterrupted() && w.tryLock()) { // Worker中的线程没有被打断并且Worker可以获取锁，这里Worker能获取锁说明Worker是个闲置Worker，在阻塞队列里拿数据一直被阻塞，没有数据进来。如果没有获取到Worker锁，说明Worker还在执行任务，不进行中断(shutdown方法不会中断正在执行的任务)
                    try {
                        t.interrupt();  // 中断Worker线程
                    } catch (SecurityException ignore) {
                    } finally {
                        w.unlock(); // 释放Worker锁
                    }
                }
                if (onlyOne) // 如果只打断1个Worker的话，直接break退出，否则，遍历所有的Worker
                    break;
            }
        } finally {
            mainLock.unlock(); // 解锁
        }
    }
    


shutdown方法将线程池状态改成SHUTDOWN，线程池还能继续处理阻塞队列里的任务，并且会回收一些闲置的Worker。但是shutdownNow方法不一样，它会把线程池状态改成STOP状态，这样不会处理阻塞队列里的任务，也不会处理新的任务：

	// shutdownNow方法会有返回值的，返回的是一个任务列表，而shutdown方法没有返回值
	public List<Runnable> shutdownNow() {
        List<Runnable> tasks;
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock(); // shutdownNow操作也需要加锁，防止并发
        try {
            checkShutdownAccess(); // 检查关闭线程池的权限
            advanceRunState(STOP); // 把线程池状态更新到STOP
			interruptWorkers(); // 中断Worker的运行
            tasks = drainQueue();
        } finally {
            mainLock.unlock(); // 解锁
        }
        tryTerminate(); // 尝试结束线程池，上面已经分析过了
        return tasks;
    }

shutdownNow的中断和shutdown方法不一样，调用的是interruptWorkers方法：

	private void interruptWorkers() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock(); // 中断Worker需要加锁，防止并发
        try {
            for (Worker w : workers)
                w.interruptIfStarted(); // 中断Worker的执行
        } finally {
            mainLock.unlock(); // 解锁
        }
    }

Worker的interruptIfStarted方法中断Worker的执行：

	void interruptIfStarted() {
       Thread t;
       // Worker无论是否被持有锁，只要还没被中断，那就中断Worker
       if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
           try {
               t.interrupt(); // 强行中断Worker的执行
           } catch (SecurityException ignore) {
           }
       }
    }

    
线程池关闭总结：    

线程池的关闭主要是两个方法，shutdown和shutdownNow方法。

shutdown方法会更新状态到SHUTDOWN，不会影响阻塞队列里任务的执行，但是不会执行新进来的任务。同时也会回收闲置的Worker，闲置Worker的定义上面已经说过了。

shutdownNow方法会更新状态到STOP，会影响阻塞队列的任务执行，也不会执行新进来的任务。同时会回收所有的Worker。
    
