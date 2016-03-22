title: Flume运行过程源码分析
date: 2015-07-07 23:23:23
tags:
- big data
- flume
categories:
- flume
description: Flume运行的主类是org.apache.flume.node.Application的main方法 ...

---------------

  
## Flume运行过程分析 ##

### 入口Application ###

Flume运行的主类是org.apache.flume.node.Application的main方法。

命令行args参数如下：

	--conf-file YourConfigFile --name AgentName

main方法里会先判断参数中是否有z或者zkConnString，如果配置了这两个参数中其中一个的话，那么会使用PollingZooKeeperConfigurationProvider或StaticZooKeeperConfigurationProvider配置。否则使用PollingPropertiesFileConfigurationProvider或PropertiesFileConfigurationProvider配置。

一般我们只传--conf-file和--name，那么就会使用PollingPropertiesFileConfigurationProvider进行配置。

	// EventBus，监听者设计模式
	EventBus eventBus = new EventBus(agentName + "-event-bus");
    // 构造PollingPropertiesFileConfigurationProvider，会处理配置文件相关的信息
	PollingPropertiesFileConfigurationProvider configurationProvider =
            new PollingPropertiesFileConfigurationProvider(
              agentName, configurationFile, eventBus, 30);
	components.add(configurationProvider);
    application = new Application(components);
    // EventBus注册application
    eventBus.register(application);
    
    ...
    
    application.start();
    
Application内部有个handleConfigurationEvent方法，使用Subscribe注解，EventBus中被使用到：

	@Subscribe
    public synchronized void handleConfigurationEvent(MaterializedConfiguration conf) {
      stopAllComponents();
      startAllComponents(conf);
    }

接下来看Application的start方法：

	public synchronized void start() {
    	// 使用生命周期组件管理器进行管理，这里其实只有1个组件，那就是PollingPropertiesFileConfigurationProvider
        for(LifecycleAware component : components) {
          // 管理器管理各个组件的时候会传入2个参数，分别是管理策略(自动重启策略)和所需状态(START状态)
          supervisor.supervise(component,
              new SupervisorPolicy.AlwaysRestartPolicy(), LifecycleState.START);
        }
    }
    
LifecycleSupervisor生命周期管理者的supervise方法的主要代码：

	
    // Supervisoree是一个带有status和policy这2个属性的封装类。这2个属性分别表示状态和管理策略。其中管理策略只有2种策略，分别是AlwaysRestartPolicy(自动重启策略)和OnceOnlyPolicy(只启动一次策略)；状态表示这个组件状态，状态Status中包括首次发生，最后发生，失败次数，目标状态等属性。
	Supervisoree process = new Supervisoree();
    process.status = new Status();

    process.policy = policy;
    // 所需状态初始化成START
    process.status.desiredState = desiredState;
    process.status.error = false;

 	// 起一个监控线程。 将Supervisoree和组件参数传入
    MonitorRunnable monitorRunnable = new MonitorRunnable();
    monitorRunnable.lifecycleAware = lifecycleAware;
    monitorRunnable.supervisoree = process;
    monitorRunnable.monitorService = monitorService;

    supervisedProcesses.put(lifecycleAware, process);

    ScheduledFuture<?> future = monitorService.scheduleWithFixedDelay(
        monitorRunnable, 0, 3, TimeUnit.SECONDS);
    monitorFutures.put(lifecycleAware, future);


监控线程MonitorRunnable的run方法主要代码：

	switch (supervisoree.status.desiredState) {
    	  // 所需状态之前已经初始化成START状态，那么会执行组件的start方法。这里的组件是之前分析的PollingPropertiesFileConfigurationProvider
          case START:
            try {
              lifecycleAware.start();
            } catch (Throwable e) {
              logger.error("Unable to start " + lifecycleAware
                  + " - Exception follows.", e);
              if (e instanceof Error) {
                // This component can never recover, shut it down.
                supervisoree.status.desiredState = LifecycleState.STOP;
                try {
                  lifecycleAware.stop();
                  logger.warn("Component {} stopped, since it could not be"
                      + "successfully started due to missing dependencies",
                      lifecycleAware);
                } catch (Throwable e1) {
                  logger.error("Unsuccessful attempt to "
                      + "shutdown component: {} due to missing dependencies."
                      + " Please shutdown the agent"
                      + "or disable this component, or the agent will be"
                      + "in an undefined state.", e1);
                  supervisoree.status.error = true;
                  if (e1 instanceof Error) {
                    throw (Error) e1;
                  }
                  // Set the state to stop, so that the conf poller can
                  // proceed.
                }
              }
              supervisoree.status.failures++;
            }
            break;
          case STOP:
            try {
              lifecycleAware.stop();
            } catch (Throwable e) {
              logger.error("Unable to stop " + lifecycleAware
                  + " - Exception follows.", e);
              if (e instanceof Error) {
                throw (Error) e;
              }
              supervisoree.status.failures++;
            }
            break;
          default:
            logger.warn("I refuse to acknowledge {} as a desired state",
                supervisoree.status.desiredState);
        }
        
PollingPropertiesFileConfigurationProvider的start方法：

	public void start() {
        LOGGER.info("Configuration provider starting");

        Preconditions.checkState(file != null,
            "The parameter file must not be null");

		// 初始化线程池
        executorService = Executors.newSingleThreadScheduledExecutor(
                new ThreadFactoryBuilder().setNameFormat("conf-file-poller-%d")
                    .build());

		// 起一个文件观察线程
        FileWatcherRunnable fileWatcherRunnable =
            new FileWatcherRunnable(file, counterGroup);

        executorService.scheduleWithFixedDelay(fileWatcherRunnable, 0, interval,
            TimeUnit.SECONDS);
		// 初始化组件的状态为START
        lifecycleState = LifecycleState.START;

        LOGGER.debug("Configuration provider started");
    }
    
文件观察线程FileWatcherRunnable的run方法：

	public void run() {
      LOGGER.debug("Checking file:{} for changes", file);

      counterGroup.incrementAndGet("file.checks");
	  // 得到文件的上次修改时间
      long lastModified = file.lastModified();
	  // 如果修改了文件，那么会执行以下代码。首次发生的时候lastChange为0，所以肯定会执行一次。以后只要配置改了才会再次执行
      if (lastModified > lastChange) {
        LOGGER.info("Reloading configuration file:{}", file);

        counterGroup.incrementAndGet("file.loads");

        lastChange = lastModified;

        try {
          // eventBus属性之前在Application中分析过，而且它注册了application实例
          // Application中有个handleConfigurationEvent方法，eventBus是个观察者设计模式，所以会钓鱼handleConfigurationEvent这个方法
          // getConfiguration方法会parse配置文件中的配置信息
          eventBus.post(getConfiguration());
        } catch (Exception e) {
          LOGGER.error("Failed to load configuration data. Exception follows.",
              e);
        } catch (NoClassDefFoundError e) {
          LOGGER.error("Failed to start agent because dependencies were not " +
              "found in classpath. Error follows.", e);
        } catch (Throwable t) {
          // caught because the caller does not handle or log Throwables
          LOGGER.error("Unhandled error", t);
        }
      }
    }
    
getConfiguration方法是在PollingPropertiesFileConfigurationProvider的父类AbstractConfigurationProvider中定义的：

	public MaterializedConfiguration getConfiguration() {
        MaterializedConfiguration conf = new SimpleMaterializedConfiguration();
        FlumeConfiguration fconfig = getFlumeConfiguration();
        AgentConfiguration agentConf = fconfig.getConfigurationFor(getAgentName());
        if (agentConf != null) {
          // 构造Channel
          Map<String, ChannelComponent> channelComponentMap = Maps.newHashMap();
          // 构造Source
          Map<String, SourceRunner> sourceRunnerMap = Maps.newHashMap();
          // 构造Sink
          Map<String, SinkRunner> sinkRunnerMap = Maps.newHashMap();
          try {
            loadChannels(agentConf, channelComponentMap);
            loadSources(agentConf, channelComponentMap, sourceRunnerMap);
            loadSinks(agentConf, channelComponentMap, sinkRunnerMap);
            Set<String> channelNames =
                new HashSet<String>(channelComponentMap.keySet());
            for(String channelName : channelNames) {
              ChannelComponent channelComponent = channelComponentMap.
                  get(channelName);
              if(channelComponent.components.isEmpty()) {
                LOGGER.warn(String.format("Channel %s has no components connected" +
                    " and has been removed.", channelName));
                channelComponentMap.remove(channelName);
                Map<String, Channel> nameChannelMap = channelCache.
                    get(channelComponent.channel.getClass());
                if(nameChannelMap != null) {
                  nameChannelMap.remove(channelName);
                }
              } else {
                LOGGER.info(String.format("Channel %s connected to %s",
                    channelName, channelComponent.components.toString()));
                conf.addChannel(channelName, channelComponent.channel);
              }
            }
            for(Map.Entry<String, SourceRunner> entry : sourceRunnerMap.entrySet()) {
              conf.addSourceRunner(entry.getKey(), entry.getValue());
            }
            for(Map.Entry<String, SinkRunner> entry : sinkRunnerMap.entrySet()) {
              conf.addSinkRunner(entry.getKey(), entry.getValue());
            }
          } catch (InstantiationException ex) {
            LOGGER.error("Failed to instantiate component", ex);
          } finally {
            channelComponentMap.clear();
            sourceRunnerMap.clear();
            sinkRunnerMap.clear();
          }
        } else {
          LOGGER.warn("No configuration found for this host:{}", getAgentName());
        }
        return conf;
    }
    
当所有的组件，Source，Channel，Sink加载完之后。会调用handleConfigurationEvent方法：

	@Subscribe
    public synchronized void handleConfigurationEvent(MaterializedConfiguration conf) {
      // 先关闭之前启动的所有Source，Channel，Sink组件(首次运行的时候并没有任何开启的组件)
      stopAllComponents();
      // 再重新开启这些组件
      startAllComponents(conf);
    }

看下startAllComponents方法：

	// 启动各个组件的时候同样适用supervisor的supervise方法。然后启动MonitorRunnable线程调用各个组件的start方法。
	private void startAllComponents(MaterializedConfiguration materializedConfiguration) {
        logger.info("Starting new configuration:{}", materializedConfiguration);

        this.materializedConfiguration = materializedConfiguration;

        for (Entry<String, Channel> entry :
          materializedConfiguration.getChannels().entrySet()) {
          try{
            logger.info("Starting Channel " + entry.getKey());
            supervisor.supervise(entry.getValue(),
                new SupervisorPolicy.AlwaysRestartPolicy(), LifecycleState.START);
          } catch (Exception e){
            logger.error("Error while starting {}", entry.getValue(), e);
          }
        }

        /*
         * Wait for all channels to start.
         */
        for(Channel ch: materializedConfiguration.getChannels().values()){
          while(ch.getLifecycleState() != LifecycleState.START
              && !supervisor.isComponentInErrorState(ch)){
            try {
              logger.info("Waiting for channel: " + ch.getName() +
                  " to start. Sleeping for 500 ms");
              Thread.sleep(500);
            } catch (InterruptedException e) {
              logger.error("Interrupted while waiting for channel to start.", e);
              Throwables.propagate(e);
            }
          }
        }

        for (Entry<String, SinkRunner> entry : materializedConfiguration.getSinkRunners()
            .entrySet()) {
          try{
            logger.info("Starting Sink " + entry.getKey());
            supervisor.supervise(entry.getValue(),
              new SupervisorPolicy.AlwaysRestartPolicy(), LifecycleState.START);
          } catch (Exception e) {
            logger.error("Error while starting {}", entry.getValue(), e);
          }
        }

        for (Entry<String, SourceRunner> entry : materializedConfiguration
            .getSourceRunners().entrySet()) {
          try{
            logger.info("Starting Source " + entry.getKey());
            supervisor.supervise(entry.getValue(),
              new SupervisorPolicy.AlwaysRestartPolicy(), LifecycleState.START);
          } catch (Exception e) {
            logger.error("Error while starting {}", entry.getValue(), e);
          }
        }

        this.loadMonitoring();
    }
    
### Flume启动过程总结 ###

首先通过Application类加载配置文件，加载后调用Application的start方法。

Application的start方法内部会针对PollingPropertiesFileConfigurationProvider组件适用组件管理者管理这个组件。也就是使用LifecycleSupervisor的supervise方法。

supervise方法内部使用MonitorRunnable监控线程。  监控线程内部会调用组件的start方法。 即PollingPropertiesFileConfigurationProvider的start方法。

PollingPropertiesFileConfigurationProvider的start方法会使用FileWatcherRunnable文件查看进程判断配置文件是否已经修改，修改的话重新加载配置文件信息，然后通过EventBus调用Application的handleConfigurationEvent方法关闭目前正在启动的组件，关闭之后重新开启组件。

各个新开启的组件会做类似的工作，使用LifecycleSupervisor的supervise方法，也就是起各个MonitorRunnable监控线程启动各个组件的start方法。


## Source组件的构造过程 ##

直接看AbstractConfigurationProvider的loadSources方法：

	private void loadSources(AgentConfiguration agentConf,
      Map<String, ChannelComponent> channelComponentMap,
      Map<String, SourceRunner> sourceRunnerMap)
      throws InstantiationException {

    Map<String, Context> sourceContexts = agentConf.getSourceContext();
        for (String sourceName : sourceNames) {
          Context context = sourceContexts.get(sourceName);
          if(context != null){
            // 使用sourceFactory构造Source
            Source source =
                sourceFactory.create(sourceName,
                    context.getString(BasicConfigurationConstants.CONFIG_TYPE));
            try {
              Configurables.configure(source, context);
              List<Channel> sourceChannels = new ArrayList<Channel>();
              String[] channelNames = context.getString(
                  BasicConfigurationConstants.CONFIG_CHANNELS).split("\\s+");
              for (String chName : channelNames) {
                ChannelComponent channelComponent = channelComponentMap.get(chName);
                if(channelComponent != null) {
                  sourceChannels.add(channelComponent.channel);
                }
              }
              if(sourceChannels.isEmpty()) {
                String msg = String.format("Source %s is not connected to a " +
                    "channel",  sourceName);
                throw new IllegalStateException(msg);
              }
              Map<String, String> selectorConfig = context.getSubProperties(
                  BasicConfigurationConstants.CONFIG_SOURCE_CHANNELSELECTOR_PREFIX);

              ChannelSelector selector = ChannelSelectorFactory.create(
                  sourceChannels, selectorConfig);

              ChannelProcessor channelProcessor = new ChannelProcessor(selector);
              Configurables.configure(channelProcessor, context);
              source.setChannelProcessor(channelProcessor);
              sourceRunnerMap.put(sourceName,
                  SourceRunner.forSource(source));
              // source关联Channel    
              for(Channel channel : sourceChannels) {
                ChannelComponent channelComponent = Preconditions.
                    checkNotNull(channelComponentMap.get(channel.getName()),
                        String.format("Channel %s", channel.getName()));
                channelComponent.components.add(sourceName);
              }
            } catch (Exception e) {
              String msg = String.format("Source %s has been removed due to an " +
                  "error during configuration", sourceName);
              LOGGER.error(msg, e);
            }
          }
        }
      }
