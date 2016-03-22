title: Flume Channel组件分析
date: 2015-06-22 23:23:23
tags:
- big data
- flume
categories:
- flume
description: Flume内置了很多Channel，比如Memory Channel, JDBC Channel, Kafka Channel, File Channel等 ...

---------------

Source组件已经分析过，接下来看Channel组件。

Flume内置了很多Channel，比如Memory Channel, JDBC Channel, Kafka Channel, File Channel等。

## Channel接口 ##

Channel接口定义：

	public interface Channel extends LifecycleAware, NamedComponent {

      public void put(Event event) throws ChannelException;

      public Event take() throws ChannelException;

      public Transaction getTransaction();
    }

LifecycleAware接口和NamedComponent接口之前在分析Source的时候已经说明过。

Channel接口有3个方法，分别是put，take和getTransaction。

Channel是存储Source收集过来的数据的，所以提供put(存储Source的Event)和take(交付给Sink)方法，它也支持事务。

AbstractChannel抽象类是Channel接口的实现类，作用跟AbstractSource是Source的实现类类似。

BasicChannelSemantics是AbstractChannel的子类，是Channel的基础实现类，内部使用ThreadLocal完成事件的处理和事务功能，实现了getTransaction方法，在这个方法内部调用createTransaction得到事务对象，createTransaction也被抽象成了一个抽象方法。

BasicChannelSemantics实现了Channel接口的put和take方法，使用Transaction接口的put和take方法。

	@Override
  	public void put(Event event) throws ChannelException {
    	BasicTransactionSemantics transaction = currentTransaction.get();
	    Preconditions.checkState(transaction != null,
    	    "No transaction exists for this thread");
	    transaction.put(event);
  	}
    
    @Override
  	public Event take() throws ChannelException {
    	BasicTransactionSemantics transaction = currentTransaction.get();
    	Preconditions.checkState(transaction != null,
        	"No transaction exists for this thread");
    	return transaction.take();
	}
    
currentTransaction是个ThreadLocal：

	private ThreadLocal<BasicTransactionSemantics> currentTransaction
	      = new ThreadLocal<BasicTransactionSemantics>();
          
currentTransaction的初始化：

	@Override
	  public Transaction getTransaction() {

    	if (!initialized) {
      	synchronized (this) {
        	if (!initialized) {
          	initialize();
          	initialized = true;
        	}
      	}
    	}

    	BasicTransactionSemantics transaction = currentTransaction.get();
    	if (transaction == null || transaction.getState().equals(
            	BasicTransactionSemantics.State.CLOSED)) {
      	transaction = createTransaction(); // 抽象方法，子类实现
      	currentTransaction.set(transaction);
    	}
    	return transaction;
  	}

这样，Channel的put和take操作就由抽象类createTransaction中得到的Transaction实现。**因此，Channel对事件的各个操作由具体的Transaction实现。**

一般普通的Channel继承这个BasicChannelSemantics类即可。

## Transaction接口 ##

由于Channel内部使用了事务特性，因此有必要介绍一个Flume事务相关的结构。

Transaction接口定义：

	public interface Transaction {

    public enum TransactionState {Started, Committed, RolledBack, Closed };

      public void begin();
      
      public void commit();

      public void rollback();

      public void close();
    }
    
基本的事务封装类BasicTransactionSemantics，BasicTransactionSemantics实现了Transaction接口的4个方法，并引入了状态的概念，内部也有个state变量表示状态，Transaction接口的4个方法的实现会基于状态：

BasicTransactionSemantics构造函数会初始化状态state为State.NEW, 实现Transaction接口的begin方法如下：

	public void begin() {
        Preconditions.checkState(Thread.currentThread().getId() == initialThreadId,
            "begin() called from different thread than getTransaction()!");
        Preconditions.checkState(state.equals(State.NEW),
            "begin() called when transaction is " + state + "!");

        try {
          doBegin();
        } catch (InterruptedException e) {
          Thread.currentThread().interrupt();
          throw new ChannelException(e.toString(), e);
        }
        state = State.OPEN;
      }

begin方法需要状态为NEW才可执行。同理commit方法需要状态为OPEN，rollback需要状态为OPEN，close方法需要状态为NEW或COMPLETED。BasicTransactionSemantics又提供了put和take方法用来放事件和取事件，所以又抽象出了4个抽象方法和2个默认实现方法：

	protected void doBegin() throws InterruptedException {}
	protected abstract void doPut(Event event) throws InterruptedException;
	protected abstract Event doTake() throws InterruptedException;
	protected abstract void doCommit() throws InterruptedException;
	protected abstract void doRollback() throws InterruptedException;
	protected void doClose() {}


## Flume内置的Channel ##

### MemoryChannel ###

Memory Channel的配置如下：

    a1.channels = c1
    a1.channels.c1.type = memory
    a1.channels.c1.capacity = 10000
    a1.channels.c1.transactionCapacity = 10000

MemoryChannel继承BasicChannelSemantics。

MemoryChannel的configure读取配置信息，它的内部有个重要的属性如下：

	private LinkedBlockingDeque<Event> queue; // 阻塞事件队列

MemoryChannel内部有个MemoryTransaction类继承自BasicTransactionSemantics，这个事务对象就是MemoryChannel使用的事务对象。

	@Override
    protected BasicTransactionSemantics createTransaction() {
      return new MemoryTransaction(transCapacity, channelCounter);
    }

MemoryTransaction的属性如下：

	// 取走事件列表
	private LinkedBlockingDeque<Event> takeList;
    // 放入事件列表
    private LinkedBlockingDeque<Event> putList;
    private final ChannelCounter channelCounter;
    private int putByteCounter = 0;
    private int takeByteCounter = 0;
    
之前分析过，Channel对事件的各个操作由具体的Transaction实现，也就是由MemoryTransaction实现。

doPut方法，加入事件，doPut方法只针对Source提供过来的Event，跟Sink无关：

	@Override
    protected void doPut(Event event) throws InterruptedException {
      channelCounter.incrementEventPutAttemptCount();
      int eventByteSize = (int)Math.ceil(estimateEventSize(event)/byteCapacitySlotSize);

      if (!putList.offer(event)) { // 事件存储到MemoryTransaction的加入事件列表中
        throw new ChannelException(
          "Put queue for MemoryTransaction of capacity " +
            putList.size() + " full, consider committing more frequently, " +
            "increasing capacity or increasing thread count");
      }
      putByteCounter += eventByteSize;
    }
    
doTake方法，取走事件，doTake方法只针对Sink，负责给Sink传递Event数据：

	@Override
    protected Event doTake() throws InterruptedException {
      channelCounter.incrementEventTakeAttemptCount();
      if(takeList.remainingCapacity() == 0) {
        throw new ChannelException("Take list for MemoryTransaction, capacity " +
            takeList.size() + " full, consider committing more frequently, " +
            "increasing capacity, or increasing thread count");
      }
      if(!queueStored.tryAcquire(keepAlive, TimeUnit.SECONDS)) {
        return null;
      }
      Event event;
      synchronized(queueLock) {
        event = queue.poll(); // 取出MemoryChannel中阻塞事件队列中的事件
      }
      Preconditions.checkNotNull(event, "Queue.poll returned NULL despite semaphore " +
          "signalling existence of entry");
      takeList.put(event); // 放到MemoryTransaction的取出事件列表中

      int eventByteSize = (int)Math.ceil(estimateEventSize(event)/byteCapacitySlotSize);
      takeByteCounter += eventByteSize;

      return event;
    }

doCommit的部分代码，提交处理的事件，doCommit只处理putList，所以提交的是Source过来的数据，并交给MemoryChannel内部的阻塞队列：

	@Override
    protected void doCommit() throws InterruptedException {
      int puts = putList.size();
      int takes = takeList.size();
      synchronized(queueLock) {
        if(puts > 0 ) {
          while(!putList.isEmpty()) { // 加入事件列表中不空的话
            if(!queue.offer(putList.removeFirst())) { // 在MemoryChannel里的阻塞事件队列中加入MemoryTransaction的加入事件队列，也就是说在MemoryTransaction中新加入的所有事件全部移植到MemoryChannel中的阻塞队列中
              throw new RuntimeException("Queue add failed, this shouldn't be able to happen");
            }
          }
        }
        putList.clear();
        takeList.clear();
      }
    }
    
doRollback方法，回滚处理的事件, 回滚操作只针对Sink，回滚要交给Sink的事件到MemoryChannel的阻塞队列里：

	@Override
    protected void doRollback() {
      int takes = takeList.size();
      synchronized(queueLock) {
        Preconditions.checkState(queue.remainingCapacity() >= takeList.size(), "Not enough space in memory channel " +
            "queue to rollback takes. This should never happen, please report");
        while(!takeList.isEmpty()) {
          // MemoryTransaction处理的事件全部回滚，回到MemoryChannel的阻塞队列里。因为MemoryTransaction处理的事件数据是从MemoryChannel的阻塞队列里取走的
          queue.addFirst(takeList.removeLast());
        }
        putList.clear();
      }
      bytesRemaining.release(putByteCounter);
      putByteCounter = 0;
      takeByteCounter = 0;

      queueStored.release(takes);
      channelCounter.setChannelSize(queue.size());
    }


MemoryChannel总结：

MemoryChannel是个Channel，Channel的作用是收集Source发来的事件数据(put操作)和发送给Sink由Source发来的事件数据(take操作)。Channel内部对事件的处理使用Transaction完成。

put操作：先放到MemoryTransaction的putList列表中，MemoryTransaction做commit操作的时候写入到MemoryChannel的阻塞队列里。

take操作：先取出MemoryChannel中阻塞队列里的事件数据，然后放入MemoryTransaction里的takeList列表中，当需要回滚的时候会将takeList列表中的事件数据回滚到MemoryChannel中的阻塞队列里。commit操作不需要任何处理，因为事件已经从MemoryChannel中的阻塞队列里取出。

MemoryChannel有2个队列容量配置，分别是capacity和transactionCapacity。capacity是MemoryChannel里的阻塞队列的容量，transactionCapacity是MemoryTransaction里的putList和takeList列表容量。

## Channel总结 ##

Channel是Flume中的桥梁，连接着Source和Sink,重要性不言而喻。Flume内置也有很多的Channel。同样地，并不是所有的Channel都能符合需求。

比如MemoryChannel在内存中处理，速度很快，但是却不能持久化，当服务器挂了，Channel中的数据会丢失，这时就需要用FileChannel或JDBCChannel，但这2种Channel速度慢。因此可以实现一个自定义的Channel，美团的 http://tech.meituan.com/mt-log-system-optimization.html 地址上就说明了一些Flume可以改进和优化的地方。
