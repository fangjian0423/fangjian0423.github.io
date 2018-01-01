title: SpringBoot源码分析之SpringBoot的启动过程
date: 2017-04-30 20:33:33
tags:
- springboot
- java
- springboot源码分析
categories: springboot

----------------

SpringBoot的启动很简单，代码如下：

```java
    @SpringBootApplication
    public class MyApplication {
        public static void main(String[] args) {
            SpringApplication.run(MyApplication.class, args);
        }
    }
```

从代码上可以看出，调用了SpringApplication的静态方法run。这个run方法会构造一个SpringApplication的实例，然后再调用这里实例的run方法就表示启动SpringBoot。

因此，想要分析SpringBoot的启动过程，我们需要熟悉SpringApplication的构造过程以及SpringApplication的run方法执行过程即可。

我们以上述这段代码为例，分析SpringBoot的启动过程。

<!--more-->

## SpringApplication的构造过程


SpringApplication构造的时候内部会调用一个private方法initialize：

```java
    public SpringApplication(Object... sources) {
      initialize(sources); // sources目前是一个MyApplication的class对象
    }

    private void initialize(Object[] sources) {
      if (sources != null && sources.length > 0) {
        this.sources.addAll(Arrays.asList(sources)); // 把sources设置到SpringApplication的sources属性中，目前只是一个MyApplication类对象
      }
      this.webEnvironment = deduceWebEnvironment(); // 判断是否是web程序(javax.servlet.Servlet和org.springframework.web.context.ConfigurableWebApplicationContext都必须在类加载器中存在)，并设置到webEnvironment属性中
      // 从spring.factories文件中找出key为ApplicationContextInitializer的类并实例化后设置到SpringApplication的initializers属性中。这个过程也就是找出所有的应用程序初始化器
      setInitializers((Collection) getSpringFactoriesInstances(
          ApplicationContextInitializer.class));
      // 从spring.factories文件中找出key为ApplicationListener的类并实例化后设置到SpringApplication的listeners属性中。这个过程就是找出所有的应用程序事件监听器
      setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
      // 找出main类，这里是MyApplication类
      this.mainApplicationClass = deduceMainApplicationClass();
    }
```

ApplicationContextInitializer，应用程序初始化器，做一些初始化的工作：

```java
    public interface ApplicationContextInitializer<C extends ConfigurableApplicationContext> {
    	void initialize(C applicationContext);
    }
```

ApplicationListener，应用程序事件(ApplicationEvent)监听器：

```java
    public interface ApplicationListener<E extends ApplicationEvent> extends EventListener {
    	void onApplicationEvent(E event);
    }
```

这里的应用程序事件(ApplicationEvent)有应用程序启动事件(ApplicationStartedEvent)，失败事件(ApplicationFailedEvent)，准备事件(ApplicationPreparedEvent)等。

应用程序事件监听器跟监听事件是绑定的。比如ConfigServerBootstrapApplicationListener只跟ApplicationEnvironmentPreparedEvent事件绑定，LiquibaseServiceLocatorApplicationListener只跟ApplicationStartedEvent事件绑定，LoggingApplicationListener跟所有事件绑定等。

默认情况下，initialize方法从spring.factories文件中找出的key为ApplicationContextInitializer的类有：

1. org.springframework.boot.context.config.DelegatingApplicationContextInitializer
2. org.springframework.boot.context.ContextIdApplicationContextInitializer
3. org.springframework.boot.context.ConfigurationWarningsApplicationContextInitializer
4. org.springframework.boot.context.web.ServerPortInfoApplicationContextInitializer
5. org.springframework.boot.autoconfigure.logging.AutoConfigurationReportLoggingInitializer

key为ApplicationListener的有：

1. org.springframework.boot.context.config.ConfigFileApplicationListener
2. org.springframework.boot.context.config.AnsiOutputApplicationListener
3. org.springframework.boot.logging.LoggingApplicationListener
4. org.springframework.boot.logging.ClasspathLoggingApplicationListener
5. org.springframework.boot.autoconfigure.BackgroundPreinitializer
6. org.springframework.boot.context.config.DelegatingApplicationListener
7. org.springframework.boot.builder.ParentContextCloserApplicationListener
8. org.springframework.boot.context.FileEncodingApplicationListener
9. org.springframework.boot.liquibase.LiquibaseServiceLocatorApplicationListener

## SpringApplication的执行

分析run方法之前，先看一下SpringApplication中的一些事件和监听器概念。

首先是SpringApplicationRunListeners类和SpringApplicationRunListener类的介绍。

SpringApplicationRunListeners内部持有SpringApplicationRunListener集合和1个Log日志类。用于SpringApplicationRunListener监听器的批量执行。

SpringApplicationRunListener看名字也知道用于监听SpringApplication的run方法的执行。

它定义了5个步骤：

1. started(run方法执行的时候立马执行；对应事件的类型是ApplicationStartedEvent)
2. environmentPrepared(ApplicationContext创建之前并且环境信息准备好的时候调用；对应事件的类型是ApplicationEnvironmentPreparedEvent)
3. contextPrepared(ApplicationContext创建好并且在source加载之前调用一次；没有具体的对应事件)
4. contextLoaded(ApplicationContext创建并加载之后并在refresh之前调用；对应事件的类型是ApplicationPreparedEvent)
5. finished(run方法结束之前调用；对应事件的类型是ApplicationReadyEvent或ApplicationFailedEvent)

SpringApplicationRunListener目前只有一个实现类EventPublishingRunListener，它把监听的过程封装成了SpringApplicationEvent事件并让内部属性(属性名为multicaster)ApplicationEventMulticaster接口的实现类SimpleApplicationEventMulticaster广播出去，广播出去的事件对象会被SpringApplication中的listeners属性进行处理。

所以说SpringApplicationRunListener和ApplicationListener之间的关系是通过ApplicationEventMulticaster广播出去的SpringApplicationEvent所联系起来的。

![](http://7x2wh6.com1.z0.glb.clouddn.com/startup2.jpg)

SpringApplication的run方法代码如下：

```java
    public ConfigurableApplicationContext run(String... args) {
      StopWatch stopWatch = new StopWatch(); // 构造一个任务执行观察器
      stopWatch.start(); // 开始执行，记录开始时间
      ConfigurableApplicationContext context = null;
      configureHeadlessProperty();
      // 获取SpringApplicationRunListeners，内部只有一个EventPublishingRunListener
      SpringApplicationRunListeners listeners = getRunListeners(args);
      // 上面分析过，会封装成SpringApplicationEvent事件然后广播出去给SpringApplication中的listeners所监听
      // 这里接受ApplicationStartedEvent事件的listener会执行相应的操作
      listeners.started();
      try {
        // 构造一个应用程序参数持有类
        ApplicationArguments applicationArguments = new DefaultApplicationArguments(
            args);
        // 创建Spring容器
        context = createAndRefreshContext(listeners, applicationArguments);
        // 容器创建完成之后执行额外一些操作
        afterRefresh(context, applicationArguments);
        // 广播出ApplicationReadyEvent事件给相应的监听器执行
        listeners.finished(context, null);
        stopWatch.stop(); // 执行结束，记录执行时间
        if (this.logStartupInfo) {
          new StartupInfoLogger(this.mainApplicationClass)
              .logStarted(getApplicationLog(), stopWatch);
        }
        return context; // 返回Spring容器
      }
      catch (Throwable ex) {
        handleRunFailure(context, listeners, ex); // 这个过程报错的话会执行一些异常操作、然后广播出ApplicationFailedEvent事件给相应的监听器执行
        throw new IllegalStateException(ex);
      }
    }
```

创建容器的方法createAndRefreshContext如下：

```java
    private ConfigurableApplicationContext createAndRefreshContext(
        SpringApplicationRunListeners listeners,
        ApplicationArguments applicationArguments) {
      ConfigurableApplicationContext context; // 定义Spring容器
      // 创建应用程序的环境信息。如果是web程序，创建StandardServletEnvironment；否则，创建StandardEnvironment
      ConfigurableEnvironment environment = getOrCreateEnvironment();
      // 配置一些环境信息。比如profile，命令行参数
      configureEnvironment(environment, applicationArguments.getSourceArgs());
      // 广播出ApplicationEnvironmentPreparedEvent事件给相应的监听器执行
      listeners.environmentPrepared(environment);
      // 环境信息的校对
      if (isWebEnvironment(environment) && !this.webEnvironment) {
        environment = convertToStandardEnvironment(environment);
      }

      if (this.bannerMode != Banner.Mode.OFF) { // 是否在控制台上打印自定义的banner
        printBanner(environment);
      }

      // Create, load, refresh and run the ApplicationContext
      context = createApplicationContext(); // 创建Spring容器
      context.setEnvironment(environment); // 设置Spring容器的环境信息
      postProcessApplicationContext(context); // 回调方法，Spring容器创建之后做一些额外的事
      applyInitializers(context); // SpringApplication的的初始化器开始工作
      // 遍历调用SpringApplicationRunListener的contextPrepared方法。目前只是将这个事件广播器注册到Spring容器中
      listeners.contextPrepared(context);
      if (this.logStartupInfo) {
        logStartupInfo(context.getParent() == null);
        logStartupProfileInfo(context);
      }

      // 把应用程序参数持有类注册到Spring容器中，并且是一个单例
      context.getBeanFactory().registerSingleton("springApplicationArguments",
          applicationArguments);

      Set<Object> sources = getSources();
      Assert.notEmpty(sources, "Sources must not be empty");
      load(context, sources.toArray(new Object[sources.size()]));
      // 广播出ApplicationPreparedEvent事件给相应的监听器执行
      listeners.contextLoaded(context);

      // Spring容器的刷新
      refresh(context);
      if (this.registerShutdownHook) {
        try {
          context.registerShutdownHook();
        }
        catch (AccessControlException ex) {
          // Not allowed in some environments.
        }
      }
      return context;
    }
```

Spring容器的创建createApplicationContext方法如下：

```java
    protected ConfigurableApplicationContext createApplicationContext() {
      Class<?> contextClass = this.applicationContextClass;
      if (contextClass == null) {
        try {
          // 如果是web程序，那么构造org.springframework.boot.context.embedded.AnnotationConfigEmbeddedWebApplicationContext容器
          // 否则构造org.springframework.context.annotation.AnnotationConfigApplicationContext容器
          contextClass = Class.forName(this.webEnvironment
              ? DEFAULT_WEB_CONTEXT_CLASS : DEFAULT_CONTEXT_CLASS);
        }
        catch (ClassNotFoundException ex) {
          throw new IllegalStateException(
              "Unable create a default ApplicationContext, "
                  + "please specify an ApplicationContextClass",
              ex);
        }
      }
      return (ConfigurableApplicationContext) BeanUtils.instantiate(contextClass);
    }
```

Spring容器创建之后有个回调方法postProcessApplicationContext：

```java
    protected void postProcessApplicationContext(ConfigurableApplicationContext context) {
  		if (this.webEnvironment) { // 如果是web程序
  			if (context instanceof ConfigurableWebApplicationContext) { // 并且也是Spring Web容器
  				ConfigurableWebApplicationContext configurableContext = (ConfigurableWebApplicationContext) context;
  				if (this.beanNameGenerator != null) { // 如果SpringApplication设置了是实例命名生成器，注册到Spring容器中
  					configurableContext.getBeanFactory().registerSingleton(
  							AnnotationConfigUtils.CONFIGURATION_BEAN_NAME_GENERATOR,
  							this.beanNameGenerator);
  				}
  			}
  		}
  		if (this.resourceLoader != null) { // 如果SpringApplication设置了资源加载器，设置到Spring容器中
  			if (context instanceof GenericApplicationContext) {
  				((GenericApplicationContext) context)
  						.setResourceLoader(this.resourceLoader);
  			}
  			if (context instanceof DefaultResourceLoader) {
  				((DefaultResourceLoader) context)
  						.setClassLoader(this.resourceLoader.getClassLoader());
  			}
  		}
  	}
```

初始化器做的工作，比如ContextIdApplicationContextInitializer会设置应用程序的id；AutoConfigurationReportLoggingInitializer会给应用程序添加一个条件注解解析器报告等：

```java
    protected void applyInitializers(ConfigurableApplicationContext context) {
      // 遍历每个初始化器，对调用对应的initialize方法
      for (ApplicationContextInitializer initializer : getInitializers()) {
        Class<?> requiredType = GenericTypeResolver.resolveTypeArgument(
            initializer.getClass(), ApplicationContextInitializer.class);
        Assert.isInstanceOf(requiredType, context, "Unable to call initializer.");
        initializer.initialize(context);
      }
    }
```

Spring容器的刷新refresh方法内部会做很多很多的事情：比如BeanFactory的设置，BeanFactoryPostProcessor接口的执行、BeanPostProcessor接口的执行、自动化配置类的解析、条件注解的解析、国际化的初始化等等。这部分内容会在之后的文章中进行讲解。

run方法中的Spring容器创建完成之后会调用afterRefresh方法，代码如下：

```java
    protected void afterRefresh(ConfigurableApplicationContext context,
        ApplicationArguments args) {
      afterRefresh(context, args.getSourceArgs()); // 目前是个空实现
      callRunners(context, args); // 调用Spring容器中的ApplicationRunner和CommandLineRunner接口的实现类
    }

    private void callRunners(ApplicationContext context, ApplicationArguments args) {
  		List<Object> runners = new ArrayList<Object>();
      // 找出Spring容器中ApplicationRunner接口的实现类
  		runners.addAll(context.getBeansOfType(ApplicationRunner.class).values());
      // 找出Spring容器中CommandLineRunner接口的实现类
  		runners.addAll(context.getBeansOfType(CommandLineRunner.class).values());
      // 对runners进行排序
  		AnnotationAwareOrderComparator.sort(runners);
      // 遍历runners依次执行
  		for (Object runner : new LinkedHashSet<Object>(runners)) {
  			if (runner instanceof ApplicationRunner) { // 如果是ApplicationRunner，进行ApplicationRunner的run方法调用
  				callRunner((ApplicationRunner) runner, args);
  			}
  			if (runner instanceof CommandLineRunner) { // 如果是CommandLineRunner，进行CommandLineRunner的run方法调用
  				callRunner((CommandLineRunner) runner, args);
  			}
  		}
  	}
```

这样run方法执行完成之后。Spring容器也已经初始化完成，各种监听器和初始化器也做了相应的工作。

## 总结

SpringBoot启动的时候，不论调用什么方法，都会构造一个SpringApplication的实例，然后调用这个实例的run方法，这样就表示启动SpringBoot。

在run方法调用之前，也就是构造SpringApplication的时候会进行初始化的工作，初始化的时候会做以下几件事：

1. 把参数sources设置到SpringApplication属性中，这个sources可以是任何类型的参数。本文的例子中这个sources就是MyApplication的class对象
2. 判断是否是web程序，并设置到webEnvironment这个boolean属性中
3. 找出所有的初始化器，默认有5个，设置到initializers属性中
4. 找出所有的应用程序监听器，默认有9个，设置到listeners属性中
5. 找出运行的主类(main class)

SpringApplication构造完成之后调用run方法，启动SpringApplication，run方法执行的时候会做以下几件事：

1. 构造一个StopWatch，观察SpringApplication的执行
2. 找出所有的SpringApplicationRunListener并封装到SpringApplicationRunListeners中，用于监听run方法的执行。监听的过程中会封装成事件并广播出去让初始化过程中产生的应用程序监听器进行监听
3. 构造Spring容器(ApplicationContext)，并返回
3.1 创建Spring容器的判断是否是web环境，是的话构造AnnotationConfigEmbeddedWebApplicationContext，否则构造AnnotationConfigApplicationContext
3.2 初始化过程中产生的初始化器在这个时候开始工作
3.3 Spring容器的刷新(完成bean的解析、各种processor接口的执行、条件注解的解析等等)
4. 从Spring容器中找出ApplicationRunner和CommandLineRunner接口的实现类并排序后依次执行


## 例子

写了一个例子用来验证分析的启动逻辑，包括自定义的初始化器、监听器、ApplicationRunner和CommandLineRunner。

地址在：https://github.com/fangjian0423/springboot-analysis/tree/master/springboot-startup
