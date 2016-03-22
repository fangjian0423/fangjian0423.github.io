title: Flume Transaction介绍
date: 2016-01-03 17:35:53
tags:
- big data
- flume
categories:
- flume
description: Flume中有一个Transaction的概念。本文仅分析Transaction的实现类MemoryTransaction的实现原理，JdbcTransaction的原理跟数据库中的Transaction类似 ...

--------------

Flume中有一个Transaction的概念。本文仅分析Transaction的实现类MemoryTransaction的实现原理，JdbcTransaction的原理跟数据库中的Transaction类似。

Transaction接口定义如下：

	public void begin();
    
    public void commit();
    
    public void rollback();
    
    public void close();
    
Transaction跟数据库中的Transaction概念类似，都有begin，commit，rollback，close方法。

Flume中的Transaction是在Channel中使用的，主要用来处理Source数据进入Channel的过程和Channel中的数据被Sink处理的过程。下面会从这2个方面根据源码分析Transaction的原理。

在分析具体的事务操作之前，看一下MemoryTransaction中的各个方法实现原理。

首先先看一下事务的获取方法：

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
		// currentTransaction是一个ThreadLocal对象
        BasicTransactionSemantics transaction = currentTransaction.get();
        // 如果是第一次获取事务或者当前事务的已经close。那么会重新create一个新的事务
        if (transaction == null || transaction.getState().equals(
                BasicTransactionSemantics.State.CLOSED)) {
          transaction = createTransaction();
          currentTransaction.set(transaction);
        }
        return transaction;
    }
    
**再重复一下，第一次拿事务或者事务关闭之后，才会重新去构造一个新的事务。各个线程之间的事务都是独立的**

MemoryTransaction是MemoryChannel中的一个内部类。

然后介绍一下MemoryTransaction和MemoryChannel中的几个重要属性。

MemoryTransaction中有2个阻塞队列，分别是putList和takeList。putList放Source进来的数据，Sink从MemoryChannel中的queue中拿数据，然后这个数据丢到takeList中。

MemoryChannel中有个阻塞队列queue。每次事务commit的时候都会把putList中的数据丢到queue中。

begin方法MemoryTransaction没做任何处理，就不分析了。

put方法：

    @Override
    protected void doPut(Event event) throws InterruptedException {
      channelCounter.incrementEventPutAttemptCount();
      int eventByteSize = (int)Math.ceil(estimateEventSize(event)/byteCapacitySlotSize);

      if (!putList.offer(event)) {
        throw new ChannelException(
          "Put queue for MemoryTransaction of capacity " +
            putList.size() + " full, consider committing more frequently, " +
            "increasing capacity or increasing thread count");
      }
      putByteCounter += eventByteSize;
    }

put方法把数据丢入putList中，这个也就是之前分析的putList这个属性的作用，putList放Source进来的数据。

commit方法的关键性代码：

    @Override
    protected void doCommit() throws InterruptedException {
      int puts = putList.size();
      int takes = takeList.size();
      synchronized(queueLock) {
        if(puts > 0 ) {
          // 清空putList，丢到外部类MemoryChannel中的queue队列里
          while(!putList.isEmpty()) {
            // MemoryChannel中的queue队列
            if(!queue.offer(putList.removeFirst())) {
              throw new RuntimeException("Queue add failed, this shouldn't be able to happen");
            }
          }
        }
        putList.clear();
        takeList.clear();
      }
    }

rollback方法关键性代码：

    @Override
    protected void doRollback() {
      int takes = takeList.size();
      synchronized(queueLock) {
        Preconditions.checkState(queue.remainingCapacity() >= takeList.size(), "Not enough space in memory channel " +
            "queue to rollback takes. This should never happen, please report");
        // 把takeList中的数据放回到queue中
        while(!takeList.isEmpty()) {
          queue.addFirst(takeList.removeLast());
        }
        putList.clear();
      }
    }
    
发生异常后才会调用rollback方法。也就是说take方法被调用之后，由于take方法是从queue中拿数据，并且放到takeList里。所以回滚的时候需要把takeList中的数据还给queue。

MemoryTransaction的close方法只是把状态改成了CLOSED，其他没做什么，就不分析了。

MemoryTransaction的take方法：

take方法从queue中拉出数据，然后放到takeList中。

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
        event = queue.poll();
      }
      Preconditions.checkNotNull(event, "Queue.poll returned NULL despite semaphore " +
          "signalling existence of entry");
      takeList.put(event);

      int eventByteSize = (int)Math.ceil(estimateEventSize(event)/byteCapacitySlotSize);
      takeByteCounter += eventByteSize;

      return event;
    }


Source数据进入Channel过程中Transaction的处理过程：

ChannelProcessor处理这个过程：

    for (Channel reqChannel : reqChannelQueue.keySet()) {
      // 获取事务
      Transaction tx = reqChannel.getTransaction();
      Preconditions.checkNotNull(tx, "Transaction object must not be null");
      try {
        // 事务开始
        tx.begin();
		// 获取Source处理的一个批次中的所有Event
        List<Event> batch = reqChannelQueue.get(reqChannel);

        for (Event event : batch) {
          // MemoryChannel的put方法会MemoryTransaction的put方法。
          reqChannel.put(event);
        }
		// 提交事务
        tx.commit();
      } catch (Throwable t) {
      	// 发生异常回滚事务
        tx.rollback();
        if (t instanceof Error) {
          LOG.error("Error while writing to required channel: " +
              reqChannel, t);
          throw (Error) t;
        } else {
          throw new ChannelException("Unable to put batch on required " +
              "channel: " + reqChannel, t);
        }
      } finally {
        if (tx != null) {
          // 最后结束事务
          tx.close();
        }
      }
    }

Channel中的数据被Sink处理的过程：

以hdfs sink为例讲解：

    public Status process() throws EventDeliveryException {
        Channel channel = getChannel();
        Transaction transaction = channel.getTransaction();
        List<BucketWriter> writers = Lists.newArrayList();
        transaction.begin();
        try {
          int txnEventCount = 0;
          for (txnEventCount = 0; txnEventCount < batchSize; txnEventCount++) {
            Event event = channel.take();
            if (event == null) {
              break;
            }
            
	      ... 

          transaction.commit();

          if (txnEventCount < 1) {
            return Status.BACKOFF;
          } else {
            sinkCounter.addToEventDrainSuccessCount(txnEventCount);
            return Status.READY;
          }
        } catch (IOException eIO) {
          transaction.rollback();
          LOG.warn("HDFS IO error", eIO);
          return Status.BACKOFF;
        } catch (Throwable th) {
          transaction.rollback();
          LOG.error("process failed", th);
          if (th instanceof Error) {
            throw (Error) th;
          } else {
            throw new EventDeliveryException(th);
          }
        } finally {
          transaction.close();
        }
 	}
    
也是一样的流程，begin，take，commit or rollback，close。

总结：

1. MemoryTransaction是MemoryChannel中的一个内部类，内部有2个阻塞队列putList和takeList。MemoryChannel内部有个queue阻塞队列。

2. putList接收Source交给Channel的event数据，takeList保存Channel交给Sink的event数据。

3. 如果是Source交给Channel任务完成，进行commit的时候。会把putList中的所有event放到MemoryChannel中的queue。

4. 如果是Source交给Channel任务失败，进行rollback的时候。程序就不会继续走下去，比如KafkaSource需要commitOffsets，如果任务失败就不会commitOffsets。

5. 如果是Sink处理完Channel带来的event，进行commit的时候。会清空takeList中的event数据，因为已经没consume。

6. 如果是Sink处理Channel带来的event失败的话，进行rollback的时候。会把takeList中的event写回到queue中。

缺点：

Flume的Transaction跟数据库的Transaction不一样。数据库中的事务回滚之后所有操作的数据都会进行处理。而Flume的却不能还原。比如HDFSSink写数据到HDFS的时候需要rollback，比如本来要写入10000条数据，但是写到5000条的时候rollback，那么已经写入的5000条数据不能回滚，而那10000条数据回到了阻塞队列里，下次再写入的时候还会重新写入这10000条数据。这样就多了5000条重复数据，这是flume设计上的缺陷。
