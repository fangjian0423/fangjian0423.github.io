title: SpringBatch中的retry和skip机制实现分析
date: 2016-11-09 08:15:22
tags:
- springbatch
- spring
- springboot
categories: spring

----------------



[SpringBatch](http://projects.spring.io/spring-batch/)是spring框架下的一个子模块，用于处理批处理的批次框架。

本文主要分析SpringBatch中的retry和skip机制的实现。

先简单说明下SpringBatch在SpringBoot中的使用。

如果要在springboot中使用batch的话，直接加入以下依赖即可：

	<dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-batch</artifactId>
    </dependency>

然后使用注解开启Batch模块：

	...
	@EnableBatchProcessing
	public class Application { ... }

之后就可以注入JobBuilderFactory和StepBuilderFactory：

	@Autowired
    private JobBuilderFactory jobs;

    @Autowired
    private StepBuilderFactory steps;

有了这2个factory之后，就可以build job。

<!--more-->

SpringBatch中的相关基础概念比如ItemReader、ItemWriter、Chunk等本文就不介绍了。

我们以FlatFileItemReader作为reader，一个自定义Writer用于打印reader中读取出来的数据。

这个定义的writer遇到good job这条数据的时候会报错，具体逻辑如下：

	@Override
    public void write(List<? extends String> items) throws Exception {
        System.out.println("handle start =====" + items);
        for(String a : items) {
            if(a.equals("good job")) {
                throw new Exception("custom exception");
            }
        }
        System.out.println("handle end.. -----" + items);
    }

其中reader中读取的文件中的数据如下：

	hello world
	hello coder
	good job
	cool
	66666
	
我们使用StepBuilderFactory构造Step，chunkSize设置为2。然后在job1中使用并执行：

	stepBuilderFactory.get("test-step").chunk(2).reader(reader).writer(writer).build();

	
执行job1后console打印如下：

	handle start =====[hello world, hello coder]
	handle end.. -----[hello world, hello coder]
	handle start =====[good job, cool]

job1遇到了good job这条数据，writer抛出了异常，由于没有使用skip或者retry机制，导致整个流程停止。job1的处理流程底层在**SimpleChunkProcessor**这个类中完成，包括processor、writer的使用。


接下里我们构造一个job2，job2使用skip机制(其中skipLimit必须要和skip(Class<? extends Throwable> type)一起使用)，skip机制可以防止writer发生异常后不停止整个job，但是需要同时满足skip的限制次数和skip对应的Exception是发生异常的父类或自身关系条件才不会停止整个job，这里我们使用Exception作为异常和Integer.MAX_VALUE作为skip的限制次数为例：

	stepBuilderFactory.get.get("test-step").chunk(2).reader(reader).writer(writer).faultTolerant().skipLimit(Integer.MAX_VALUE).skip(Exception.class).build();
	
执行job2	后console打印如下：

	handle start =====[hello world, hello coder]
	handle end.. -----[hello world, hello coder]
	handle start =====[good job, cool]
	handle start =====[good job]
	handle start =====[cool]
	handle end.. -----[cool]
	handle start =====[66666]
	handle end.. -----[66666]
	
我们看到good job这条数据发生的异常被skip掉了，job完整的执行。

但是发现了另外一个问题，那就是处理 [good job, cool] 这批数据的时候，发生了异常，但是接下来执行了 [good job] 和 [cool] 这两批chunk为1的批次。这是在ItemWriter中执行的，它也会在ItemWriteListener中执行多次。

**换句话说，如果使用了skip功能，那么对于需要被skip的批次数据中会进行scan操作找出具体是哪1条数据的原因，这里的scan操作指的是一条一条数据的遍历。**

这个过程为什么叫scan呢?  在源码中，FaultTolerantChunkProcessor类(处理带有skip或者retry机制的处理器，跟SimpleChunkProcessor类似，只不过SimpleChunkProcessor处理简单的Job)里有个私有方法scan：

	private void scan(final StepContribution contribution, final Chunk<I> inputs, final Chunk<O> outputs,
			ChunkMonitor chunkMonitor, boolean recovery) throws Exception {

		...

		Chunk<I>.ChunkIterator inputIterator = inputs.iterator();
		Chunk<O>.ChunkIterator outputIterator = outputs.iterator();

		List<O> items = Collections.singletonList(outputIterator.next()); // 拿出需要写的数据中的每一条数据
		inputIterator.next();
		try {
			writeItems(items); // 写每条数据
			doAfterWrite(items);
			contribution.incrementWriteCount(1);
			inputIterator.remove();
			outputIterator.remove();
		}
		catch (Exception e) { // 写的时候如果发生了异常
			doOnWriteError(e, items);
			if (!shouldSkip(itemWriteSkipPolicy, e, -1) && !rollbackClassifier.classify(e)) {
				inputIterator.remove();
				outputIterator.remove();
			}
			else {
				// 具体的skip策略
				checkSkipPolicy(inputIterator, outputIterator, e, contribution, recovery);
			}
			if (rollbackClassifier.classify(e)) {
				throw e;
			}
		}
		chunkMonitor.incrementOffset();
		if (outputs.isEmpty()) { // 批次里的所有数据处理完毕之后 scanning 设置为false
			data.scanning(false);
			inputs.setBusy(false);
			chunkMonitor.resetOffset();
		}
	}

这个scan方法触发的条件是UserData这个内部类里的scanning被设置为true，这里被设置为true是在处理批次数据出现异常后并且不能retry的情况下才会被设置的。

	try {
		batchRetryTemplate.execute(retryCallback, recoveryCallback, new DefaultRetryState(inputs,
				rollbackClassifier));
	}
	catch (Exception e) {
		RetryContext context = contextHolder.get();
	 	if (!batchRetryTemplate.canRetry(context)) {
	 		// 设置scanning为true
			data.scanning(true);
		}
		throw e;
	}

这就是为什么skip机制在skip数据的时候会去scan批次中的每条数据，然后并找出需要被skip的数据的原理。


job3带有retry功能，retry的功能在于出现某个异常并且这个异常可以被retry所接受的话会进行retry，retry的次数可以进行配置，我们配置了3次retry：
	
	stepBuilderFactory.get.get("test-step").chunk(2).reader(reader).writer(writer).faultTolerant().skipLimit(Integer.MAX_VALUE).skip(Exception.class).retryLimit(3).retry(Exception.class).build();
	
	
执行 job3后console打印如下：
	
	handle start =====[hello world, hello coder]
	handle end.. -----[hello world, hello coder]
	handle start =====[good job, cool]
	handle start =====[good job, cool]
	handle start =====[good job, cool]
	handle start =====[good job]
	handle start =====[cool]
	handle end.. -----[cool]
	handle start =====[66666]
	handle end.. -----[66666]
	
[good job, cool] 这批数据retry了3次，而且都失败了。失败之后进行了skip操作。

SpringBatch中的retry和skip都有对应的policy实现，默认的retry policy是SimpleRetryPolicy，可以设置retry次数和接收的exception。比如可以使用NeverRetryPolicy：

	.retryPolicy(new NeverRetryPolicy())
	
使用NeverRetryPolicy之后，便不再retry了，只会skip。SpringBatch内部的retry是使用Spring的[retry模块](https://github.com/spring-projects/spring-retry)完成的。执行的时候可以设置RetryCallback和RecoveryCallback。


SpringBatch中默认的skip policy是LimitCheckingItemSkipPolicy。




参考资料: 

http://stackoverflow.com/questions/16567432/how-is-the-skipping-implemented-in-spring-batch

http://docs.spring.io/spring-batch/reference/html/retry.html

https://github.com/spring-projects/spring-retry