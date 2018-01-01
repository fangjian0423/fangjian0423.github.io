title: Java阻塞队列ArrayBlockingQueue和LinkedBlockingQueue实现原理分析
date: 2016-05-10 20:30:57
tags:
- java
categories: java

----------------

Java中的阻塞队列接口BlockingQueue继承自Queue接口。

BlockingQueue接口提供了3个添加元素方法。

1. add：添加元素到队列里，添加成功返回true，由于容量满了添加失败会抛出IllegalStateException异常
2. offer：添加元素到队列里，添加成功返回true，添加失败返回false
3. put：添加元素到队列里，如果容量满了会阻塞直到容量不满

3个删除方法。

1. poll：删除队列头部元素，如果队列为空，返回null。否则返回元素。
2. remove：基于对象找到对应的元素，并删除。删除成功返回true，否则返回false
3. take：删除队列头部元素，如果队列为空，一直阻塞到队列有元素并删除

常用的阻塞队列具体类有ArrayBlockingQueue、LinkedBlockingQueue、PriorityBlockingQueue、LinkedBlockingDeque等。

本文以ArrayBlockingQueue和LinkedBlockingQueue为例，分析它们的实现原理。

<!--more-->


## ArrayBlockingQueue ##

ArrayBlockingQueue的原理就是使用一个可重入锁和这个锁生成的两个条件对象进行并发控制(classic two-condition algorithm)。

ArrayBlockingQueue是一个带有长度的阻塞队列，初始化的时候必须要指定队列长度，且指定长度之后不允许进行修改。

它带有的属性如下：

	// 存储队列元素的数组，是个循环数组
    final Object[] items;

    // 拿数据的索引，用于take，poll，peek，remove方法
    int takeIndex;

    // 放数据的索引，用于put，offer，add方法
    int putIndex;

	// 元素个数
    int count;

    // 可重入锁
    final ReentrantLock lock;
	// notEmpty条件对象，由lock创建
    private final Condition notEmpty;
	// notFull条件对象，由lock创建
    private final Condition notFull;

### 数据的添加 ###

ArrayBlockingQueue有不同的几个数据添加方法，add、offer、put方法。

add方法：

	public boolean add(E e) {
        if (offer(e))
            return true;
        else
            throw new IllegalStateException("Queue full");
    }

add方法内部调用offer方法如下：

	public boolean offer(E e) {
        checkNotNull(e); // 不允许元素为空
        final ReentrantLock lock = this.lock;
        lock.lock(); // 加锁，保证调用offer方法的时候只有1个线程
        try {
            if (count == items.length) // 如果队列已满
                return false; // 直接返回false，添加失败
            else {
                insert(e); // 数组没满的话调用insert方法
                return true; // 返回true，添加成功
            }
        } finally {
            lock.unlock(); // 释放锁，让其他线程可以调用offer方法
        }
    }
    
insert方法如下：

	private void insert(E x) {
        items[putIndex] = x; // 元素添加到数组里
        putIndex = inc(putIndex); // 放数据索引+1，当索引满了变成0
        ++count; // 元素个数+1
        notEmpty.signal(); // 使用条件对象notEmpty通知，比如使用take方法的时候队列里没有数据，被阻塞。这个时候队列insert了一条数据，需要调用signal进行通知
    }
    
put方法：

	public void put(E e) throws InterruptedException {
        checkNotNull(e); // 不允许元素为空
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly(); // 加锁，保证调用put方法的时候只有1个线程
        try {
            while (count == items.length) // 如果队列满了，阻塞当前线程，并加入到条件对象notFull的等待队列里
                notFull.await(); // 线程阻塞并被挂起，同时释放锁
            insert(e); // 调用insert方法
        } finally {
            lock.unlock(); // 释放锁，让其他线程可以调用put方法
        }
    }
    
ArrayBlockingQueue的添加数据方法有add，put，offer这3个方法，总结如下：

add方法内部调用offer方法，如果队列满了，抛出IllegalStateException异常，否则返回true

offer方法如果队列满了，返回false，否则返回true

add方法和offer方法不会阻塞线程，put方法如果队列满了会阻塞线程，直到有线程消费了队列里的数据才有可能被唤醒。

这3个方法内部都会使用可重入锁保证原子性。

### 数据的删除 ###

ArrayBlockingQueue有不同的几个数据删除方法，poll、take、remove方法。

poll方法：

	public E poll() {
        final ReentrantLock lock = this.lock;
        lock.lock(); // 加锁，保证调用poll方法的时候只有1个线程
        try {
            return (count == 0) ? null : extract(); // 如果队列里没元素了，返回null，否则调用extract方法
        } finally {
            lock.unlock(); // 释放锁，让其他线程可以调用poll方法
        }
    }
    
poll方法内部调用extract方法：
    
    private E extract() {
        final Object[] items = this.items;
        E x = this.<E>cast(items[takeIndex]); // 得到取索引位置上的元素
        items[takeIndex] = null; // 对应取索引上的数据清空
        takeIndex = inc(takeIndex); // 取数据索引+1，当索引满了变成0
        --count; // 元素个数-1
        notFull.signal(); // 使用条件对象notFull通知，比如使用put方法放数据的时候队列已满，被阻塞。这个时候消费了一条数据，队列没满了，就需要调用signal进行通知
        return x; // 返回元素
    }

take方法：

	public E take() throws InterruptedException {
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly(); // 加锁，保证调用take方法的时候只有1个线程
        try {
            while (count == 0) // 如果队列空，阻塞当前线程，并加入到条件对象notEmpty的等待队列里
                notEmpty.await(); // 线程阻塞并被挂起，同时释放锁
            return extract(); // 调用extract方法
        } finally {
            lock.unlock(); // 释放锁，让其他线程可以调用take方法
        }
    }
    
remove方法：
    
    public boolean remove(Object o) {
        if (o == null) return false;
        final Object[] items = this.items;
        final ReentrantLock lock = this.lock;
        lock.lock(); // 加锁，保证调用remove方法的时候只有1个线程
        try {
            for (int i = takeIndex, k = count; k > 0; i = inc(i), k--) { // 遍历元素
                if (o.equals(items[i])) { // 两个对象相等的话
                    removeAt(i); // 调用removeAt方法
                    return true; // 删除成功，返回true
                }
            }
            return false; // 删除成功，返回false
        } finally {
            lock.unlock(); // 释放锁，让其他线程可以调用remove方法
        }
    }

removeAt方法：

	void removeAt(int i) {
        final Object[] items = this.items;
        if (i == takeIndex) { // 如果要删除数据的索引是取索引位置，直接删除取索引位置上的数据，然后取索引+1即可
            items[takeIndex] = null;
            takeIndex = inc(takeIndex);
        } else { // 如果要删除数据的索引不是取索引位置，移动元素元素，更新取索引和放索引的值
            for (;;) {
                int nexti = inc(i);
                if (nexti != putIndex) {
                    items[i] = items[nexti];
                    i = nexti;
                } else {
                    items[i] = null;
                    putIndex = i;
                    break;
                }
            }
        }
        --count; // 元素个数-1
        notFull.signal(); // 使用条件对象notFull通知，比如使用put方法放数据的时候队列已满，被阻塞。这个时候消费了一条数据，队列没满了，就需要调用signal进行通知 
    }

ArrayBlockingQueue的删除数据方法有poll，take，remove这3个方法，总结如下：

poll方法对于队列为空的情况，返回null，否则返回队列头部元素。

remove方法取的元素是基于对象的下标值，删除成功返回true，否则返回false。

poll方法和remove方法不会阻塞线程。

take方法对于队列为空的情况，会阻塞并挂起当前线程，直到有数据加入到队列中。

这3个方法内部都会调用notFull.signal方法通知正在等待队列满情况下的阻塞线程。


## LinkedBlockingQueue ##


LinkedBlockingQueue是一个使用链表完成队列操作的阻塞队列。链表是单向链表，而不是双向链表。

内部使用放锁和拿锁，这两个锁实现阻塞("two lock queue" algorithm)。

它带有的属性如下：

	// 容量大小
    private final int capacity;

    // 元素个数，因为有2个锁，存在竞态条件，使用AtomicInteger
    private final AtomicInteger count = new AtomicInteger(0);

    // 头结点
    private transient Node<E> head;

    // 尾节点
    private transient Node<E> last;

    // 拿锁
    private final ReentrantLock takeLock = new ReentrantLock();

    // 拿锁的条件对象
    private final Condition notEmpty = takeLock.newCondition();

    // 放锁
    private final ReentrantLock putLock = new ReentrantLock();

	// 放锁的条件对象
    private final Condition notFull = putLock.newCondition();
    
    
ArrayBlockingQueue只有1个锁，添加数据和删除数据的时候只能有1个被执行，不允许并行执行。

而LinkedBlockingQueue有2个锁，放锁和拿锁，添加数据和删除数据是可以并行进行的，当然添加数据和删除数据的时候只能有1个线程各自执行。

### 数据的添加 ###

LinkedBlockingQueue有不同的几个数据添加方法，add、offer、put方法。

add方法内部调用offer方法：

	public boolean offer(E e) {
        if (e == null) throw new NullPointerException(); // 不允许空元素
        final AtomicInteger count = this.count;
        if (count.get() == capacity) // 如果容量满了，返回false
            return false;
        int c = -1;
        Node<E> node = new Node(e); // 容量没满，以新元素构造节点
        final ReentrantLock putLock = this.putLock;
        putLock.lock(); // 放锁加锁，保证调用offer方法的时候只有1个线程
        try {
            if (count.get() < capacity) { // 再次判断容量是否已满，因为可能拿锁在进行消费数据，没满的话继续执行
                enqueue(node); // 节点添加到链表尾部
                c = count.getAndIncrement(); // 元素个数+1
                if (c + 1 < capacity) // 如果容量还没满
                    notFull.signal(); // 在放锁的条件对象notFull上唤醒正在等待的线程，表示可以再次往队列里面加数据了，队列还没满
            }
        } finally {
            putLock.unlock(); // 释放放锁，让其他线程可以调用offer方法
        }
        if (c == 0) // 由于存在放锁和拿锁，这里可能拿锁一直在消费数据，count会变化。这里的if条件表示如果队列中还有1条数据
            signalNotEmpty(); // 在拿锁的条件对象notEmpty上唤醒正在等待的1个线程，表示队列里还有1条数据，可以进行消费
        return c >= 0; // 添加成功返回true，否则返回false
    }
    
put方法：

	public void put(E e) throws InterruptedException {
        if (e == null) throw new NullPointerException(); // 不允许空元素
        int c = -1;
        Node<E> node = new Node(e); // 以新元素构造节点
        final ReentrantLock putLock = this.putLock;
        final AtomicInteger count = this.count;
        putLock.lockInterruptibly(); // 放锁加锁，保证调用put方法的时候只有1个线程
        try {
            while (count.get() == capacity) { // 如果容量满了
                notFull.await(); // 阻塞并挂起当前线程
            }
            enqueue(node); // 节点添加到链表尾部
            c = count.getAndIncrement(); // 元素个数+1
            if (c + 1 < capacity) // 如果容量还没满
                notFull.signal(); // 在放锁的条件对象notFull上唤醒正在等待的线程，表示可以再次往队列里面加数据了，队列还没满
        } finally {
            putLock.unlock(); // 释放放锁，让其他线程可以调用put方法
        }
        if (c == 0) // 由于存在放锁和拿锁，这里可能拿锁一直在消费数据，count会变化。这里的if条件表示如果队列中还有1条数据
            signalNotEmpty(); // 在拿锁的条件对象notEmpty上唤醒正在等待的1个线程，表示队列里还有1条数据，可以进行消费
    }
    
LinkedBlockingQueue的添加数据方法add，put，offer跟ArrayBlockingQueue一样，不同的是它们的底层实现不一样。

ArrayBlockingQueue中放入数据阻塞的时候，需要消费数据才能唤醒。

而LinkedBlockingQueue中放入数据阻塞的时候，因为它内部有2个锁，可以并行执行放入数据和消费数据，不仅在消费数据的时候进行唤醒插入阻塞的线程，同时在插入的时候如果容量还没满，也会唤醒插入阻塞的线程。

### 数据的删除 ###

LinkedBlockingQueue有不同的几个数据删除方法，poll、take、remove方法。

poll方法：

	public E poll() {
        final AtomicInteger count = this.count;
        if (count.get() == 0) // 如果元素个数为0
            return null; // 返回null
        E x = null;
        int c = -1;
        final ReentrantLock takeLock = this.takeLock;
        takeLock.lock(); // 拿锁加锁，保证调用poll方法的时候只有1个线程
        try {
            if (count.get() > 0) { // 判断队列里是否还有数据
                x = dequeue(); // 删除头结点
                c = count.getAndDecrement(); // 元素个数-1
                if (c > 1) // 如果队列里还有元素
                    notEmpty.signal(); // 在拿锁的条件对象notEmpty上唤醒正在等待的线程，表示队列里还有数据，可以再次消费
            }
        } finally {
            takeLock.unlock(); // 释放拿锁，让其他线程可以调用poll方法
        }
        if (c == capacity) // 由于存在放锁和拿锁，这里可能放锁一直在添加数据，count会变化。这里的if条件表示如果队列中还可以再插入数据
            signalNotFull(); // 在放锁的条件对象notFull上唤醒正在等待的1个线程，表示队列里还能再次添加数据
                    return x;
    }
    
take方法：

	public E take() throws InterruptedException {
        E x;
        int c = -1;
        final AtomicInteger count = this.count;
        final ReentrantLock takeLock = this.takeLock;
        takeLock.lockInterruptibly(); // 拿锁加锁，保证调用take方法的时候只有1个线程
        try {
            while (count.get() == 0) { // 如果队列里已经没有元素了
                notEmpty.await(); // 阻塞并挂起当前线程
            }
            x = dequeue(); // 删除头结点
            c = count.getAndDecrement(); // 元素个数-1
            if (c > 1) // 如果队列里还有元素
                notEmpty.signal(); // 在拿锁的条件对象notEmpty上唤醒正在等待的线程，表示队列里还有数据，可以再次消费
        } finally {
            takeLock.unlock(); // 释放拿锁，让其他线程可以调用take方法
        }
        if (c == capacity) // 由于存在放锁和拿锁，这里可能放锁一直在添加数据，count会变化。这里的if条件表示如果队列中还可以再插入数据
            signalNotFull(); // 在放锁的条件对象notFull上唤醒正在等待的1个线程，表示队列里还能再次添加数据
        return x;
    }

remove方法：

	public boolean remove(Object o) {
        if (o == null) return false;
        fullyLock(); // remove操作要移动的位置不固定，2个锁都需要加锁
        try {
            for (Node<E> trail = head, p = trail.next; // 从链表头结点开始遍历
                 p != null;
                 trail = p, p = p.next) {
                if (o.equals(p.item)) { // 判断是否找到对象
                    unlink(p, trail); // 修改节点的链接信息，同时调用notFull的signal方法
                    return true;
                }
            }
            return false;
        } finally {
            fullyUnlock(); // 2个锁解锁
        }
    }

LinkedBlockingQueue的take方法对于没数据的情况下会阻塞，poll方法删除链表头结点，remove方法删除指定的对象。

需要注意的是remove方法由于要删除的数据的位置不确定，需要2个锁同时加锁。
