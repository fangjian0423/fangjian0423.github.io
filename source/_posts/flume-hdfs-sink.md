title: 通过源码分析Flume HDFSSink 写hdfs文件的过程
date: 2015-07-20 23:32:33
tags:
- big data
- flume
categories:
- flume
description: 分析Flume HDFSSink写hdfs文件过程

---------------

Flume有HDFS Sink，可以将Source进来的数据写入到hdfs中。

HDFS Sink具体的逻辑代码是在HDFSEventSink这个类中。

HDFS Sink跟写文件相关的配置如下：

hdfs.path -> hdfs目录路径
hdfs.filePrefix -> 文件前缀。默认值FlumeData
hdfs.fileSuffix -> 文件后缀
hdfs.rollInterval -> 多久时间后close hdfs文件。单位是秒，默认30秒。设置为0的话表示不根据时间close hdfs文件
hdfs.rollSize -> 文件大小超过一定值后，close文件。默认值1024，单位是字节。设置为0的话表示不基于文件大小
hdfs.rollCount -> 写入了多少个事件后close文件。默认值是10个。设置为0的话表示不基于事件个数
hdfs.fileType -> 文件格式， 有3种格式可选择：SequenceFile, DataStream or CompressedStream
hdfs.batchSize -> 批次数，HDFS Sink每次从Channel中拿的事件个数。默认值100
hdfs.minBlockReplicas -> HDFS每个块最小的replicas数字，不设置的话会取hadoop中的配置
hdfs.maxOpenFiles -> 允许最多打开的文件数，默认是5000。如果超过了这个值，越早的文件会被关闭
serializer -> HDFS Sink写文件的时候会进行序列化操作。会调用对应的Serializer借口，可以自定义符合需求的Serializer
hdfs.retryInterval -> 关闭HDFS文件失败后重新尝试关闭的延迟数，单位是秒
hdfs.callTimeout -> HDFS操作允许的时间，比如hdfs文件的open，write，flush，close操作。单位是毫秒，默认值是10000

## HDFSEventSink分析 ##

以一个hdfs.path，hdfs.filePrefix和hdfs.fileSuffix分别为/data/%Y/%m/%d/%H，flume, .txt 为例子，分析源码：

直接看HDFSEventSink的process方法：

	public Status process() throws EventDeliveryException {
    	// 得到Channel
        Channel channel = getChannel();
        Transaction transaction = channel.getTransaction();
        // 构造一个BucketWriter集合，BucketWriter就是处理hdfs文件的具体逻辑实现类
        List<BucketWriter> writers = Lists.newArrayList();
        // Channel的事务启动
        transaction.begin();
        try {
          int txnEventCount = 0;
          // 每次处理batchSize个事件。这里的batchSize就是之前配置的hdfs.batchSize
          for (txnEventCount = 0; txnEventCount < batchSize; txnEventCount++) {
            Event event = channel.take();
            if (event == null) {
              break;
            }

            // 构造hdfs文件所在的路径
            String realPath = BucketPath.escapeString(filePath, event.getHeaders(),
                timeZone, needRounding, roundUnit, roundValue, useLocalTime);
            // 构造hdfs文件名, fileName就是之前配置的hdfs.filePrefix，即flume
            String realName = BucketPath.escapeString(fileName, event.getHeaders(),
              timeZone, needRounding, roundUnit, roundValue, useLocalTime);

			// 构造hdfs文件路径，根据之前的path，filePrefix，fileSuffix
            // 得到这里的lookupPath为 /data/2015/07/20/15/flume
            String lookupPath = realPath + DIRECTORY_DELIMITER + realName;
            BucketWriter bucketWriter;
            HDFSWriter hdfsWriter = null;
            // 构造一个回调函数
            WriterCallback closeCallback = new WriterCallback() {
              @Override
              public void run(String bucketPath) {
                LOG.info("Writer callback called.");
                synchronized (sfWritersLock) {
                  // sfWriters是一个HashMap，最多支持maxOpenFiles个键值对。超过maxOpenFiles的话会关闭越早进来的文件
                  // 回调函数的作用就是hdfs文件close的时候移除sfWriters中对应的那个文件。防止打开的文件数超过maxOpenFiles
                  // sfWriters这个Map中的key是要写的hdfs路径，value是BucketWriter
                  sfWriters.remove(bucketPath);
                }
              }
            };
            synchronized (sfWritersLock) {
              // 先查看sfWriters是否已经存在key为/data/2015/07/20/15/flume的BucketWriter
              bucketWriter = sfWriters.get(lookupPath);
              if (bucketWriter == null) {
              	// 没有的话构造一个BucketWriter
                // 先根据fileType得到对应的HDFSWriter，fileType默认有3种类型，分别是SequenceFile, DataStream or CompressedStream
                hdfsWriter = writerFactory.getWriter(fileType);
                // 构造一个BucketWriter，会将刚刚构造的hdfsWriter当做参数传入，BucketWriter写hdfs文件的时候会使用HDFSWriter
                bucketWriter = initializeBucketWriter(realPath, realName,
                  lookupPath, hdfsWriter, closeCallback);
                // 新构造的BucketWriter放入到sfWriters中
                sfWriters.put(lookupPath, bucketWriter);
              }
            }

            // 将BucketWriter放入到writers集合中
            if (!writers.contains(bucketWriter)) {
              writers.add(bucketWriter);
            }

            // 写hdfs数据
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

          // 每个批次全部完成后flush所有的hdfs文件
          for (BucketWriter bucketWriter : writers) {
            bucketWriter.flush();
          }
			
          // 事务提交
          transaction.commit();

          if (txnEventCount < 1) {
            return Status.BACKOFF;
          } else {
            sinkCounter.addToEventDrainSuccessCount(txnEventCount);
            return Status.READY;
          }
        } catch (IOException eIO) {
          // 发生异常事务回滚
          transaction.rollback();
          LOG.warn("HDFS IO error", eIO);
          return Status.BACKOFF;
        } catch (Throwable th) {
          // 发生异常事务回滚
          transaction.rollback();
          LOG.error("process failed", th);
          if (th instanceof Error) {
            throw (Error) th;
          } else {
            throw new EventDeliveryException(th);
          }
        } finally {
          // 关闭事务
          transaction.close();
        }
    }

## BucketWriter分析 ##

接下来我们看下BucketWriter的append和flush方法。

append方法：

	  public synchronized void append(final Event event)
          throws IOException, InterruptedException {
        checkAndThrowInterruptedException();
        // If idleFuture is not null, cancel it before we move forward to avoid a
        // close call in the middle of the append.
        if(idleFuture != null) {
          idleFuture.cancel(false);
          // There is still a small race condition - if the idleFuture is already
          // running, interrupting it can cause HDFS close operation to throw -
          // so we cannot interrupt it while running. If the future could not be
          // cancelled, it is already running - wait for it to finish before
          // attempting to write.
          if(!idleFuture.isDone()) {
            try {
              idleFuture.get(callTimeout, TimeUnit.MILLISECONDS);
            } catch (TimeoutException ex) {
              LOG.warn("Timeout while trying to cancel closing of idle file. Idle" +
                " file close may have failed", ex);
            } catch (Exception ex) {
              LOG.warn("Error while trying to cancel closing of idle file. ", ex);
            }
          }
          idleFuture = null;
        }

        // 如果hdfs文件没有被打开
        if (!isOpen) {
          // hdfs已关闭的话抛出异常
          if (closed) {
            throw new BucketClosedException("This bucket writer was closed and " +
              "this handle is thus no longer valid");
          }
          // 打开hdfs文件
          open();
        }

        // 查看是否需要创建新文件
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
          	// 如果需要创建新文件的时候会关闭文件，然后再打开新的文件。这里的close方法没有参数，表示可以再次打开新的文件
            close();
            open();
          }
        }

		// 写event数据
        try {
          sinkCounter.incrementEventDrainAttemptCount();
          callWithTimeout(new CallRunner<Void>() {
            @Override
            public Void call() throws Exception {
              // 真正的写数据使用HDFSWriter的append方法
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

        // 文件大小+起来
        processSize += event.getBody().length;
        // 事件个数+1
        eventCounter++;
        // 批次数+1
        batchCounter++;

		// 批次数达到配置的hdfs.batchSize的话调用flush方法
        if (batchCounter == batchSize) {
          flush();
        }
    }
    
先看下open方法，打开hdfs文件的方法：

	private void open() throws IOException, InterruptedException {
    	// hdfs文件路径或HDFSWriter没构造的话抛出异常
        if ((filePath == null) || (writer == null)) {
          throw new IOException("Invalid file settings");
        }

        final Configuration config = new Configuration();
        // disable FileSystem JVM shutdown hook
        config.setBoolean("fs.automatic.close", false);

        // Hadoop is not thread safe when doing certain RPC operations,
        // including getFileSystem(), when running under Kerberos.
        // open() must be called by one thread at a time in the JVM.
        // NOTE: tried synchronizing on the underlying Kerberos principal previously
        // which caused deadlocks. See FLUME-1231.
        synchronized (staticLock) {
          checkAndThrowInterruptedException();

          try {
          	// fileExtensionCounter是一个AtomicLong类型的实例，初始化为当前时间戳的数值
            // 由于之前分析的，可能存在先关闭文件，然后再次open新文件的情况。所以在同一个BucketWriter类中open方法得到的文件名时间戳仅仅相差1
            // 得到时间戳counter
            long counter = fileExtensionCounter.incrementAndGet();

			// 最终的文件名加上时间戳，这就是为什么flume生成的文件名会带有时间戳的原因
            // 这里的fullFileName就是 flume.1437375933234
            String fullFileName = fileName + "." + counter;

			// 加上后缀名， fullFileName就成了flume.1437375933234.txt
            if (fileSuffix != null && fileSuffix.length() > 0) {
              fullFileName += fileSuffix;
            } else if (codeC != null) {
              fullFileName += codeC.getDefaultExtension();
            }

			// 由于没配置inUsePrefix和inUseSuffix。 故这两个属性的值分别为""和".tmp"
            // buckerPath为 /data/2015/07/20/15/flume.1437375933234.txt.tmp
            bucketPath = filePath + "/" + inUsePrefix
              + fullFileName + inUseSuffix;
            // targetPath为 /data/2015/07/20/15/flume.1437375933234.txt
            targetPath = filePath + "/" + fullFileName;

            LOG.info("Creating " + bucketPath);
            callWithTimeout(new CallRunner<Void>() {
              @Override
              public Void call() throws Exception {
                if (codeC == null) {
                  // Need to get reference to FS using above config before underlying
                  // writer does in order to avoid shutdown hook &
                  // IllegalStateExceptions
                  if(!mockFsInjected) {
                    fileSystem = new Path(bucketPath).getFileSystem(
                      config);
                  }
                  // 使用HDFSWriter打开文件
                  writer.open(bucketPath);
                } else {
                  // need to get reference to FS before writer does to
                  // avoid shutdown hook
                  if(!mockFsInjected) {
                    fileSystem = new Path(bucketPath).getFileSystem(
                      config);
                  }
                  // 使用HDFSWriter打开文件
                  writer.open(bucketPath, codeC, compType);
                }
                return null;
              }
            });
          } catch (Exception ex) {
            sinkCounter.incrementConnectionFailedCount();
            if (ex instanceof IOException) {
              throw (IOException) ex;
            } else {
              throw Throwables.propagate(ex);
            }
          }
        }
        isClosedMethod = getRefIsClosed();
        sinkCounter.incrementConnectionCreatedCount();
        // 重置各个计数器
        resetCounters();

        // 开线程处理hdfs.rollInterval配置的参数，多长时间后调用close方法
        if (rollInterval > 0) {
          Callable<Void> action = new Callable<Void>() {
            public Void call() throws Exception {
              LOG.debug("Rolling file ({}): Roll scheduled after {} sec elapsed.",
                  bucketPath, rollInterval);
              try {
                // Roll the file and remove reference from sfWriters map.
                close(true);
              } catch(Throwable t) {
                LOG.error("Unexpected error", t);
              }
              return null;
            }
          };
          // 以秒为单位在这里指定。将这个线程执行的结果赋值给timedRollFuture这个属性
          timedRollFuture = timedRollerPool.schedule(action, rollInterval,
              TimeUnit.SECONDS);
        }

        isOpen = true;
    }
    
flush方法，只会在close和append方法(处理的事件数等于批次数)中被调用：

	public synchronized void flush() throws IOException, InterruptedException {
        checkAndThrowInterruptedException();
        if (!isBatchComplete()) { //isBatchComplete判断batchCount是否等于0。 所以这里只要batchCount不为0，那么执行下去
          doFlush(); // doFlush方法会调用HDFSWriter的sync方法，并且将batchCount设置为0

		  // idleTimeout没有配置，以下代码不会执行
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

close方法：

	  public synchronized void close(boolean callCloseCallback)
		    throws IOException, InterruptedException {
        checkAndThrowInterruptedException();
        try {
          // close的时候先执行flush方法，清空batchCount，并调用HDFSWriter的sync方法
          flush();
        } catch (IOException e) {
          LOG.warn("pre-close flush failed", e);
        }
        boolean failedToClose = false;
        LOG.info("Closing {}", bucketPath);
        // 创建一个关闭线程，这个线程会调用HDFSWriter的close方法
        CallRunner<Void> closeCallRunner = createCloseCallRunner();
        if (isOpen) { // 如果文件还开着
          try {
          	// 执行HDFSWriter的close方法
            callWithTimeout(closeCallRunner);
            sinkCounter.incrementConnectionClosedCount();
          } catch (IOException e) {
            LOG.warn(
              "failed to close() HDFSWriter for file (" + bucketPath +
                "). Exception follows.", e);
            sinkCounter.incrementConnectionFailedCount();
            failedToClose = true;
            // 关闭文件失败的话起个线程，retryInterval秒后继续执行
            final Callable<Void> scheduledClose =
              createScheduledCloseCallable(closeCallRunner);
            timedRollerPool.schedule(scheduledClose, retryInterval,
              TimeUnit.SECONDS);
          }
          isOpen = false;
        } else {
          LOG.info("HDFSWriter is already closed: {}", bucketPath);
        }

        // timedRollFuture就是根据hdfs.rollInterval配置生成的一个属性。如果hdfs.rollInterval配置为0，那么不会执行以下代码
        // 因为要close文件，所以如果开启了hdfs.rollInterval等待时间到了flush文件，由于文件已经关闭，再次关闭会有问题
        // 所以这里取消timedRollFuture线程的执行
        if (timedRollFuture != null && !timedRollFuture.isDone()) {
          timedRollFuture.cancel(false); // do not cancel myself if running!
          timedRollFuture = null;
        }

		// 没有配置hdfs.idleTimeout， 不会执行
        if (idleFuture != null && !idleFuture.isDone()) {
          idleFuture.cancel(false); // do not cancel myself if running!
          idleFuture = null;
        }

        // 重命名文件，如果报错了，不会重命名文件
        if (bucketPath != null && fileSystem != null && !failedToClose) {
		  // 将 /data/2015/07/20/15/flume.1437375933234.txt.tmp 重命名为 /data/2015/07/20/15/flume.1437375933234.txt
          renameBucket(bucketPath, targetPath, fileSystem);
        }
        if (callCloseCallback) { // callCloseCallback是close方法的参数
        
          // 调用关闭文件的回调函数，也就是BucketWriter的onCloseCallback属性
          // 这个onCloseCallback属性就是在HDFSEventSink里的回调函数closeCallback。 用来处理sfWriters.remove(bucketPath);
          // 如果onCloseCallback属性为true，那么说明这个BucketWriter已经不会再次open新的文件了。生命周期已经到了。
          // onCloseCallback只有在append方法中调用shouldRotate方法的时候需要close文件的时候才会传入false，其他情况都是true
          runCloseAction(); 
          
          closed = true;
        }
    }

再回过头来看下append方法里的shouldRotate方法，shouldRotate方法执行下去的话会关闭文件然后再次打开新的文件：

	private boolean shouldRotate() {
        boolean doRotate = false;

		// 调用HDFSWriter的isUnderReplicated方法，用来判断当前hdfs文件是否正在复制。
        if (writer.isUnderReplicated()) {
          this.isUnderReplicated = true;
          doRotate = true;
        } else {
          this.isUnderReplicated = false;
        }

		// rollCount就是配置的hdfs.rollCount。 eventCounter事件数达到rollCount之后，会close文件，然后创建新的文件
        if ((rollCount > 0) && (rollCount <= eventCounter)) {
          LOG.debug("rolling: rollCount: {}, events: {}", rollCount, eventCounter);
          doRotate = true;
        }

		// rollSize就是配置的hdfs.rollSize。processSize是每个事件加起来的文件大小。当processSize超过rollSize的时候，会close文件，然后创建新的文件
        if ((rollSize > 0) && (rollSize <= processSize)) {
          LOG.debug("rolling: rollSize: {}, bytes: {}", rollSize, processSize);
          doRotate = true;
        }

        return doRotate;
    }


## HDFSWriter分析 ##

每个BucketWriter中对应只有一个HDFSWriter。

HDFSWriter是一个接口，有3个具体的实现类，分别是：HDFSDataStream，HDFSSequenceFile和HDFSCompressedDataStream。分别对应fileType为DataStream，SequenceFile和CompressedStream。

我们以HDFSDataStream为例，分析一下在BucketWriter中用到的HDFSWriter的一些方法：

append方法，写hdfs文件：

	@Override
    public void append(Event e) throws IOException {
    	// 非常简单，直接使用serializer的write方法
        // serializer是org.apache.flume.serialization.EventSerializer接口的实现类
        // 默认的Serializer是BodyTextEventSerializer
    	serializer.write(e);
    }

open方法：

	@Override
    public void open(String filePath) throws IOException {
        Configuration conf = new Configuration();
        // 构造hdfs路径
        Path dstPath = new Path(filePath);
        FileSystem hdfs = getDfs(conf, dstPath);
        // 调用doOpen方法
        doOpen(conf, dstPath, hdfs);
	}
    
	protected void doOpen(Configuration conf,
        Path dstPath, FileSystem hdfs) throws
		    IOException {
        if(useRawLocalFileSystem) {
          if(hdfs instanceof LocalFileSystem) {
            hdfs = ((LocalFileSystem)hdfs).getRaw();
          } else {
            logger.warn("useRawLocalFileSystem is set to true but file system " +
                "is not of type LocalFileSystem: " + hdfs.getClass().getName());
          }
        }

        boolean appending = false;
        // 构造FSDataOutputStream，作为属性outStream
        if (conf.getBoolean("hdfs.append.support", false) == true && hdfs.isFile
                (dstPath)) {
          outStream = hdfs.append(dstPath);
          appending = true;
        } else {
          outStream = hdfs.create(dstPath);
        }

		// 初始化Serializer
        serializer = EventSerializerFactory.getInstance(
            serializerType, serializerContext, outStream);
        if (appending && !serializer.supportsReopen()) {
          outStream.close();
          serializer = null;
          throw new IOException("serializer (" + serializerType +
              ") does not support append");
        }

        // must call superclass to check for replication issues
        registerCurrentStream(outStream, hdfs, dstPath);

        if (appending) {
          serializer.afterReopen();
        } else {
          serializer.afterCreate();
        }
    }

close方法：

	@Override
    public void close() throws IOException {
        serializer.flush();
        serializer.beforeClose();
        outStream.flush();
        outStream.sync();
        outStream.close();

        unregisterCurrentStream();
    }

sync方法：

	@Override
    public void sync() throws IOException {
        serializer.flush();
        outStream.flush();
        outStream.sync();
    }

isUnderReplicated方法，在AbstractHDFSWriter中定义：

	@Override
    public boolean isUnderReplicated() {
        try {
          // 得到目前文件replication后的块数
          int numBlocks = getNumCurrentReplicas();
          if (numBlocks == -1) {
            return false;
          }
          int desiredBlocks;
          if (configuredMinReplicas != null) {
            // 如果配置了hdfs.minBlockReplicas
            desiredBlocks = configuredMinReplicas;
          } else {
            // 没配置hdfs.minBlockReplicas的话直接从hdfs配置中拿
            desiredBlocks = getFsDesiredReplication();
          }
          // 如果当前复制的块比期望要复制的块数字要小的话，返回true
          return numBlocks < desiredBlocks;
        } catch (IllegalAccessException e) {
          logger.error("Unexpected error while checking replication factor", e);
        } catch (InvocationTargetException e) {
          logger.error("Unexpected error while checking replication factor", e);
        } catch (IllegalArgumentException e) {
          logger.error("Unexpected error while checking replication factor", e);
        }
        return false;
    }

## 总结 ##


hdfs.rollInterval，hdfs.rollSize，hdfs.rollCount，hdfs.minBlockReplicas，hdfs.batchSize这5个配置影响着hdfs文件的关闭。

**注意，这5个配置影响的是一个hdfs文件，是一个hdfs文件。当hdfs文件关闭的时候，这些配置指标会重新开始计算。因为BucketWriter中的open方法里会调用resetCounters方法，这个方法会重置计数器。而基于hdfs.rollInterval的timedRollFuture线程返回值是在close方法中被销毁的。因此，只要close文件，并且open新文件的时候，这5个属性都会重新开始计算。**

hdfs.rollInterval与时间有关，当时间达到hdfs.rollInterval配置的秒数，那么会close文件。

hdfs.rollSize与每个event的字节大小有关，当一个一个event的字节相加起来大于等于hdfs.rollSize的时候，那么会close文件。

hdfs.rollCount与事件的个数有关，当事件个数大于等于hdfs.rollCount的时候，那么会close文件。

hdfs.batchSize表示当事件添加到hdfs.batchSize个的时候，也就是说HDFS Sink每次会拿hdfs.batchSize个事件，而且这些所有的事件都写进了同一个hdfs文件，这才会触发本次条件，并且其他4个配置都未达成条件。然后会close文件。

hdfs.minBlockReplicas表示期望hdfs对文件最小的复制块数。所以有时候我们配置了hdfs.rollInterval，hdfs.rollSize，hdfs.rollCount这3个参数，并且这3个参数都没有符合条件，但是还是生成了多个文件，这就是因为这个参数导致的，而且这个参数的优先级比hdfs.rollSize，hdfs.rollCount要高。



