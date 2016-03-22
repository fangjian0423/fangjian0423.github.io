title: Flume Source组件分析
date: 2015-06-21 23:23:23
tags:
- big data
- flume
categories:
- flume
description: Flume内置了很多Source，比如Avro Source，Spooling Directory Source，NetCat Source，Kafka Source等 ...

---------------


Flume的介绍以及它的架构之前已经分析过。本文分析flume的Source组件。

Flume内置了很多Source，比如Avro Source，Spooling Directory Source，NetCat Source，Kafka Source等。

以源码的角度分析Source组件。

## Source接口 ##

Source接口的定义：

	public interface Source extends LifecycleAware, NamedComponent {

      public void setChannelProcessor(ChannelProcessor channelProcessor);

      public ChannelProcessor getChannelProcessor();
      
    }
    
Source接口只有2个方法，分别是setChannelProcessor和getChannelProcessor。ChannelProcessor是一个Channel处理器，会暴露一些操作用来将事件存储到channel中。

Source接口继承了LifecycleAware和NamedComponent接口。

LifecycleAware接口表示实现了该接口的类是具有状态的，有生命周期的特性。

	public interface LifecycleAware {

      public void start(); // 启动一个服务或组件

      public void stop(); // 关闭一个服务或组件

      public LifecycleState getLifecycleState(); // 得到当前组件或服务的状态

    }

NamedComponent接口是一个可以让组件拥有名字的接口，这样组件可以在配置中被引用。

	public interface NamedComponent {

      public void setName(String name);

      public String getName();

    }
    
Source接口的实现类AbstractSource，基本上所有的Source都会继承这个AbstractSource：

AbstractSource属性：

	private ChannelProcessor channelProcessor;
	private String name;
	private LifecycleState lifecycleState;

这3个属性是都是都会在Source接口的方法中被使用到。

	@Override
      public synchronized void start() {
        Preconditions.checkState(channelProcessor != null,
            "No channel processor configured");

        lifecycleState = LifecycleState.START;
      }

      @Override
      public synchronized void stop() {
        lifecycleState = LifecycleState.STOP;
      }

      @Override
      public synchronized void setChannelProcessor(ChannelProcessor cp) {
        channelProcessor = cp;
      }

      @Override
      public synchronized ChannelProcessor getChannelProcessor() {
        return channelProcessor;
      }

      @Override
      public synchronized LifecycleState getLifecycleState() {
        return lifecycleState;
      }

      @Override
      public synchronized void setName(String name) {
        this.name = name;
      }

      @Override
      public synchronized String getName() {
        return name;
      }

## Flume内置的Source例子 ##

### NetCat Source ###

首先看最简单的NetCat Source。

NetCat Source配置如下：

	a1.sources.r1.type = netcat
	a1.sources.r1.bind = localhost
	a1.sources.r1.port = 44444
    
NetCat Source对应的Source类是NetcatSource，定义如下：

	public class NetcatSource extends AbstractSource implements Configurable,
		    EventDrivenSource
        
EventDrivenSource接口表示这个Source不需要外部驱动来收集event，而是自身会实现。这个接口只是一个标识，没有任何方法。

Configurable接口提供了一个public void configure(Context context);方法，会使用以下Context进行一些数据的配置。

NetcatSource覆盖了AbstractSource的start和stop方法，也实现了Configurable中的configure方法。

实现configure方法：

读取配置文件中的信息host和port信息(还有其他一些属性的配置，比如max-line-length和ack-every-event)。

覆盖start方法：

内部会使用ServerSocketChannel监听对应地址和端口发来的数据。

覆盖stop方法：

关闭ServerSocketChannel。



start方法内部会用线程池起一个新的进程来监听数据，其中有个processEvents方法，processEvents方法内部会读取Socket发来的数据，有段代码：

	Event event = EventBuilder.withBody(body);
    // process event
    ChannelException ex = null;
    try {
      source.getChannelProcessor().processEvent(event);
    } catch (ChannelException chEx) {
      ex = chEx;
    }

这段代码就会处理socket读取的数据并构造成一个Event，然后放到ChannelProcessor里，这个ChannelProcessor是在AbstractSource中定义的。

### Kafka Source ###

Kafka Source也是Flume内置的一个Source之一，配置如下：

    tier1.sources.source1.type = org.apache.flume.source.kafka.KafkaSource
    tier1.sources.source1.channels = channel1
    tier1.sources.source1.zookeeperConnect = localhost:2181
    tier1.sources.source1.topic = test1
    tier1.sources.source1.groupId = flume
    tier1.sources.source1.kafka.consumer.timeout.ms = 100
    
对应的Source类是KafkaSource：

	public class KafkaSource extends AbstractSource
        implements Configurable, PollableSource
        
PollableSource接口表示Source需要自己去查询数据(poll)。

    public interface PollableSource extends Source {
      
      // process方法的返回值是个Status，有2个状态，分别是READY和BACKOFF
      public Status process() throws EventDeliveryException;

      public static enum Status {
        READY, BACKOFF
      }
    }

KafkaSource的configure方法同样是读取配置文件信息。KafkaSource有几个特殊的配置，zookeeperConnect，groupId，topic都是kafka本身需要的配置。batchSize和batchDurationMillis是批处理的配置。batchSize表示批次数，每batchSize个event需要写入到Channel中，默认值是1000。batchDurationMillis是处理Kafka对i读取数据的时间，默认是1000毫秒，比如批次个数没有到1000，但是batchDurationMillis到了的话还是会丢到channel里。

start方法构造Kafka的ConsumerConnector，start方法还会调用父类AbstractSource的start方法，也就是初始化lifecycleState属性，说明这个Source是有生命周期的。

stop方法关闭Kafka的ConsumerConnector，同理stop也会调用父类的stop方法。

process方法是PollableSource接口的方法，KafkaSource需要我们自己去查询数据。

process部分源码如下：

	long batchStartTime = System.currentTimeMillis();
    long batchEndTime = System.currentTimeMillis() + timeUpperLimit;
    try {
      boolean iterStatus = false;
      // 批次个数和处理时间全部符合才会进行处理
      while (eventList.size() < batchUpperLimit &&
              System.currentTimeMillis() < batchEndTime) {
        iterStatus = hasNext();
        if (iterStatus) {
		  // kafka队列存在数据的话，提取数据，构造Event，放到eventList里，eventList是KafkaSource里的一个属性
          MessageAndMetadata<byte[], byte[]> messageAndMetadata = it.next();
          kafkaMessage = messageAndMetadata.message();
          kafkaKey = messageAndMetadata.key();

          headers = new HashMap<String, String>();
          headers.put(KafkaSourceConstants.TIMESTAMP,
                  String.valueOf(System.currentTimeMillis()));
          headers.put(KafkaSourceConstants.TOPIC, topic);
          headers.put(KafkaSourceConstants.KEY, new String(kafkaKey));

          event = EventBuilder.withBody(kafkaMessage, headers);
          eventList.add(event);
        }

      }
      if (eventList.size() > 0) {
        // eventList列表里有数据的话，将数据丢到channel里，清空eventList。代码执行到这里说明要么读取了1000条数据，要么处理时间到了
        getChannelProcessor().processEventBatch(eventList);
        eventList.clear();
       
        if (!kafkaAutoCommitEnabled) {
          consumer.commitOffsets();
        }
      }
      // kafka队列没有数据的话返回BACKOFF状态
      if (!iterStatus) {
        return Status.BACKOFF;
      }
      // kafka队列还有数据的话返回READY状态
      return Status.READY;
    } catch (Exception e) {
      log.error("KafkaSource EXCEPTION, {}", e);
      return Status.BACKOFF;
    }


### 编写自定义的Source ###

Flume尽管已经提供了不少Source，但始终无法满足所有的需求。 比如数据库中的数据想使用Flume写入了其他渠道。

编写SQLSource只需要继承AbstractSource，且实现Configurable和PollableSource接口，SQLSource跟KafkaSource一样都需要自身去查询数据，所以都得实现PollableSource接口。

SQLSource在github上已经有人实现过了: https://github.com/keedio/flume-ng-sql-source
