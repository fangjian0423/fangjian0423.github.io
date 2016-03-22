title: Flume Sink组件分析
date: 2015-06-23 01:23:23
tags:
- big data
- flume
categories:
- flume
description: Flume内置了很多Sink，比如HDFS Sink, Hive Sink, File Roll Sink, HBase Sink, ElasticSearch Sink等 ...

---------------

Source和Channel组件已经分析过，接下来看Sink组件。

Flume内置了很多Sink，比如HDFS Sink, Hive Sink, File Roll Sink, HBase Sink, ElasticSearch Sink等。

## Sink接口 ##

	public interface Sink extends LifecycleAware, NamedComponent {
      
      public void setChannel(Channel channel);

      public Channel getChannel();

      public Status process() throws EventDeliveryException;

      public static enum Status {
        READY, BACKOFF
      }
    }
    
沒啥好讲的，跟之前的都类似。

Sink实现类AbstractSink：

	abstract public class AbstractSink implements Sink, LifecycleAware
    
跟AbstractSource类似。

## Flume内置的Sink例子 ##

### HDFSEventSink ###

1个简单的hdfs sink配置：

	a1.channels = c1
    a1.sinks = k1
    a1.sinks.k1.type = hdfs
    a1.sinks.k1.channel = c1
    a1.sinks.k1.hdfs.path = /flume/events/%y-%m-%d/%H%M/%S
    a1.sinks.k1.hdfs.filePrefix = events-
    a1.sinks.k1.hdfs.round = true
    a1.sinks.k1.hdfs.roundValue = 10
    a1.sinks.k1.hdfs.roundUnit = minute
    
HDFSEventSink中的process(Sink接口提供的)方法记录着如何写入hdfs文件：

    public Status process() throws EventDeliveryException {
    	// 得到Channel
        Channel channel = getChannel();
        // 得到Channel里的Transaction，接下来事件的提交，回滚都基于这个Transaction
        Transaction transaction = channel.getTransaction();
        List<BucketWriter> writers = Lists.newArrayList();
        // 事务开始，一般的Transaction实现类不会覆盖这个方法，除非有特殊要求，begin方法默认的实现不做任何事
        transaction.begin();
        try {
          int txnEventCount = 0;
          // 每次操作都处理batchSize个事件，batchSize可配置
          for (txnEventCount = 0; txnEventCount < batchSize; txnEventCount++) {
          	// channel的take方法内部会使用Transaction的take方法，Transaction回滚后这些take出来的事件全部都会回滚到Channel里
            Event event = channel.take();
            if (event == null) {
              break;
            }

            // reconstruct the path name by substituting place holders
            String realPath = BucketPath.escapeString(filePath, event.getHeaders(),
                timeZone, needRounding, roundUnit, roundValue, useLocalTime);
            String realName = BucketPath.escapeString(fileName, event.getHeaders(),
              timeZone, needRounding, roundUnit, roundValue, useLocalTime);

            String lookupPath = realPath + DIRECTORY_DELIMITER + realName;
            BucketWriter bucketWriter;
            HDFSWriter hdfsWriter = null;
            // Callback to remove the reference to the bucket writer from the
            // sfWriters map so that all buffers used by the HDFS file
            // handles are garbage collected.
            WriterCallback closeCallback = new WriterCallback() {
              @Override
              public void run(String bucketPath) {
                LOG.info("Writer callback called.");
                synchronized (sfWritersLock) {
                  sfWriters.remove(bucketPath);
                }
              }
            };
            synchronized (sfWritersLock) {
              bucketWriter = sfWriters.get(lookupPath);
              // we haven't seen this file yet, so open it and cache the handle
              if (bucketWriter == null) {
                hdfsWriter = writerFactory.getWriter(fileType);
                bucketWriter = initializeBucketWriter(realPath, realName,
                  lookupPath, hdfsWriter, closeCallback);
                sfWriters.put(lookupPath, bucketWriter);
              }
            }

            // track the buckets getting written in this transaction
            if (!writers.contains(bucketWriter)) {
              writers.add(bucketWriter);
            }

            // Write the data to HDFS
            try {
              bucketWriter.append(event);
            } catch (BucketClosedException ex) {
              LOG.info("Bucket was closed while trying to append, " +
                "reinitializing bucket and writing event.");
              hdfsWriter = writerFactory.getWriter(fileType);
              bucketWriter = initializeBucketWriter(realPath, realName,
                lookupPath, hdfsWriter, closeCallback);
              synchronized (sfWritersLock) {
                sfWriters.put(lookupPath, bucketWriter);
              }
              bucketWriter.append(event);
            }
          }

          if (txnEventCount == 0) {
            sinkCounter.incrementBatchEmptyCount();
          } else if (txnEventCount == batchSize) {
            sinkCounter.incrementBatchCompleteCount();
          } else {
            sinkCounter.incrementBatchUnderflowCount();
          }

          // flush all pending buckets before committing the transaction
          for (BucketWriter bucketWriter : writers) {
            bucketWriter.flush();
          }

		  // 写完数据无误后，commit，清空Transaction里的数据
          transaction.commit();

          if (txnEventCount < 1) {
            return Status.BACKOFF;
          } else {
            sinkCounter.addToEventDrainSuccessCount(txnEventCount);
            return Status.READY;
          }
        } catch (IOException eIO) {
          // 发送异常回滚
          transaction.rollback();
          LOG.warn("HDFS IO error", eIO);
          return Status.BACKOFF;
        } catch (Throwable th) {
          // 发送异常回滚
          transaction.rollback();
          LOG.error("process failed", th);
          if (th instanceof Error) {
            throw (Error) th;
          } else {
            throw new EventDeliveryException(th);
          }
        } finally {
          // 事务结束，一般的Transaction实现类不会覆盖这个方法，除非有特殊要求，close方法默认的实现不做任何事
          transaction.close();
        }
      }	

下面我们重点分析一下HDFS Sink如何写入数据到hdfs文件。

分析之前，有几个配置跟是否创建hdfs新文件有关，当达到这些配置的条件后，sink会关闭文件，然后重新创建1个新文件重复执行

	hdfs.rollInterval -> 每隔多少秒会关闭文件。默认30秒
    hdfs.rollSize -> 文件大小到达一定量后，会关闭文件。单位bytes。默认1024，如果是0的话表示跟文件大小无关
    hdfs.batchSize -> 批次数，Sink每次处理都会处理batchSize个事件, 不会创建新文件。默认100个
    hdfs.rollCount -> 事件处理了rollCount个后，会关闭文件。默认10个，如果是0的话表示跟事件个数无关
    hdfs.minBlockReplicas -> hadoop中的dfs.replication配置属性，表示复制块的个数，默认会根据hadoop的配置

HDFS Sink每个根据batchSize，遍历channel中的事件，针对每个Event的生成地址构造BucketWriter。然后在sfWriters这个Map中以文件的路径为key，BucketWriter为value进行存储。

接下来使用BucketWriter的append方法，当batchSize个事件遍历完成，调用BucketWriter的flush方法。

	bucketWriter.append(event);
    
    for (BucketWriter bucketWriter : writers) {
        bucketWriter.flush();
    }
    
BucketWriter的append方法部分代码如下：

	// 打开文件
	if (!isOpen) {
      if (closed) {
        throw new BucketClosedException("This bucket writer was closed and " +
          "this handle is thus no longer valid");
      }
      open();
    }

    // 是否需要创建新文件
    if (shouldRotate()) {
      boolean doRotate = true;

      if (isUnderReplicated) {
        if (maxConsecUnderReplRotations > 0 &&
            consecutiveUnderReplRotateCount >= maxConsecUnderReplRotations) {
          doRotate = false;
          if (consecutiveUnderReplRotateCount == maxConsecUnderReplRotations) {
            LOG.error("Hit max consecutive under-replication rotations ({}); " +
                "will not continue rolling files under this path due to " +
                "under-replication", maxConsecUnderReplRotations);
          }
        } else {
          LOG.warn("Block Under-replication detected. Rotating file.");
        }
        consecutiveUnderReplRotateCount++;
      } else {
        consecutiveUnderReplRotateCount = 0;
      }

      if (doRotate) {
        close();
        open();
      }
    }

    try {
      sinkCounter.incrementEventDrainAttemptCount();
      // 写hdfs文件
      callWithTimeout(new CallRunner<Void>() {
        @Override
        public Void call() throws Exception {
          writer.append(event); // could block
          return null;
        }
      });
    } catch (IOException e) {
      LOG.warn("Caught IOException writing to HDFSWriter ({}). Closing file (" +
          bucketPath + ") and rethrowing exception.",
          e.getMessage());
      try {
        close(true);
      } catch (IOException e2) {
        LOG.warn("Caught IOException while closing file (" +
             bucketPath + "). Exception follows.", e2);
      }
      throw e;
    }
    
    // 更新一些属性
    processSize += event.getBody().length;
    eventCounter++;
    batchCounter++;

	// 写入的事件达到批次数，flush
    if (batchCounter == batchSize) {
      flush();
    }

shouldRotate方法表示是否需要创建新文件：

    private boolean shouldRotate() {
        boolean doRotate = false;

		// 先判断HDFS是否正在复制块，优先级最高，也就是配置文件的minBlockReplicas
        if (writer.isUnderReplicated()) {
          this.isUnderReplicated = true;
          doRotate = true;
        } else {
          this.isUnderReplicated = false;
        }

		// 配置文件中的rollCount，也就是事件个数
        if ((rollCount > 0) && (rollCount <= eventCounter)) {
          LOG.debug("rolling: rollCount: {}, events: {}", rollCount, eventCounter);
          doRotate = true;
        }
		// 配置文件中的rollSize，也就文件大小
        if ((rollSize > 0) && (rollSize <= processSize)) {
          LOG.debug("rolling: rollSize: {}, bytes: {}", rollSize, processSize);
          doRotate = true;
        }

        return doRotate;
    }
    
有的同学可能因为没有配置minBlockReplicas，而配置了其他属性，所以hdfs还是会生成很多文件，这是因为minBlockReplicas的优先级最高，如果当前正在复制块，其他所有的条件都会被无视。

flush方法：

	public synchronized void flush() throws IOException, InterruptedException {
        checkAndThrowInterruptedException();
        if (!isBatchComplete()) {
          doFlush();

          if(idleTimeout > 0) {
            // if the future exists and couldn't be cancelled, that would mean it has already run
            // or been cancelled
            if(idleFuture == null || idleFuture.cancel(false)) {
              Callable<Void> idleAction = new Callable<Void>() {
                public Void call() throws Exception {
                  LOG.info("Closing idle bucketWriter {} at {}", bucketPath,
                    System.currentTimeMillis());
                  if (isOpen) {
                    close(true);
                  }
                  return null;
                }
              };
              idleFuture = timedRollerPool.schedule(idleAction, idleTimeout,
                  TimeUnit.SECONDS);
            }
          }
        }
    }

总结:

HDFSSink里的hdfs.rollSize，hdfs.rollCount，hdfs.minBlockReplicas，hdfs.rollInterval,hdfs.batchSize这些配置都是在BucketWriter中才会使用。

hdfs.rollSize, hdfs.rollCount和hdfs.minBlockReplicas, hdfs.rollInterval这4个属性决定是否创建新文件。

hdfs.batchSize决定是否flush文件数据。 由于HDFS Sink中每次最多只有batchSize个事件，因此BucketWriter会事件个数达到了batchSize后直接flush数据即可。




    
    
    
