title: SpringBoot源码分析之日志系统的构造
date: 2017-08-23 15:32:34
tags:
- java
- springboot
- springboot源码分析
categories: springboot

----------------


SpringBoot对日志的配置和加载进行了封装，让我们可以很方便地使用一些日志框架，只需要定义对应日志框架的配置文件，比如LogBack、Log4j、Log4j2等，代码内部便可以直接使用。

比如我们在resources目录下定义了一个logback.xml文件，文件内容是logback相关的配置，然后就可以直接在代码在使用Logger记录日志啦。

下图是SpringBoot对日志功能的封装：

![](http://7x2wh6.com1.z0.glb.clouddn.com/logging-system.png)


<!--more-->

## LoggingSystem内部结构

从图中也可以发现目前SpringBoot支持4种类型的日志，分别是JDK内置的Log(JavaLoggingSystem)、Log4j(Log4JLoggingSystem)、Log4j2(Log4J2LoggingSystem)以及Logback(LogbackLoggingSystem)。

LoggingSystem是个抽象类，内部有这几个方法：

1. beforeInitialize方法：日志系统初始化之前需要处理的事情。抽象方法，不同的日志架构进行不同的处理
2. initialize方法：初始化日志系统。默认不进行任何处理，需子类进行初始化工作
3. cleanUp方法：日志系统的清除工作。默认不进行任何处理，需子类进行清除工作
4. getShutdownHandler方法：返回一个Runnable用于当jvm退出的时候处理日志系统关闭后需要进行的操作，默认返回null，也就是什么都不做
5. setLogLevel方法：抽象方法，用于设置对应logger的级别


AbstractLoggingSystem抽象类继承LoggingSystem抽象类，进行了一些扩展，重点在于initialize方法：

1. 实现了beforeInitialize方法，但是内部不做任何处理
2. 复写了initialize方法，具体过程如下

```java
@Override
public void initialize(LoggingInitializationContext initializationContext,
		String configLocation, LogFile logFile) {
	// 如果传递了日志配置文件，调用initializeWithSpecificConfig方法，使用指定的文件
	if (StringUtils.hasLength(configLocation)) {
		initializeWithSpecificConfig(initializationContext, configLocation, logFile);
		return;
	}
	// 没有传递日志配置文件的话调用initializeWithConventions方法，使用约定俗成的方式
	initializeWithConventions(initializationContext, logFile);
}

private void initializeWithSpecificConfig(
		LoggingInitializationContext initializationContext, String configLocation,
		LogFile logFile) {
	// 处理日志配置文件中的占位符
	configLocation = SystemPropertyUtils.resolvePlaceholders(configLocation);
	loadConfiguration(initializationContext, configLocation, logFile);
}

private void initializeWithConventions(
		LoggingInitializationContext initializationContext, LogFile logFile) {
	// 获取自初始化的日志配置文件，该方法会使用getStandardConfigLocations抽象方法得到的文件数组
	// 然后进行遍历，如果文件存在，返回对应的文件目录。注意这里的文件指的是classpath下的文件
	String config = getSelfInitializationConfig();
	// 如果找到对应的日志配置文件并且logFile为null(logFile为null表示只有console会输出)
	if (config != null && logFile == null) {
		// 调用reinitialize方法重新初始化
		// 默认的reinitialize方法不做任何处理，logback,log4j和log4j2覆盖了这个方法，会进行处理
		reinitialize(initializationContext);
		return;
	}
	// 如果没有找到对应的日志配置文件
	if (config == null) {
		// 调用getSpringInitializationConfig方法获取日志配置文件
		// 该方法与getSelfInitializationConfig方法的区别在于getStandardConfigLocations方法得到的文件数组内部遍历的逻辑
		// getSelfInitializationConfig方法直接遍历并判断classpath下是否存在对应的文件
		// getSpringInitializationConfig方法遍历后判断的文件名会在后缀前加上 "-spring" 字符串
		// 比如查找logback.xml文件，getSelfInitializationConfig会直接查找classpath下是否存在logback.xml文件，而getSpringInitializationConfig方法会判断classpath下是否存在logback-spring.xml文件
		config = getSpringInitializationConfig();
	}
	// 如果找到了对应的日志配置文件
	if (config != null) {
		// 调用loadConfiguration抽象方法，让子类实现
		loadConfiguration(initializationContext, config, logFile);
		return;
	}
	// 还是没找到日志配置文件的话，调用loadDefaults抽象方法加载，让子类实现
	loadDefaults(initializationContext, logFile);
}

protected abstract String[] getStandardConfigLocations();

protected abstract void loadConfiguration(
		LoggingInitializationContext initializationContext, String location,
		LogFile logFile);

protected abstract void loadDefaults(
		LoggingInitializationContext initializationContext, LogFile logFile);
```

以LogbackLoggingSystem类为例，分析具体的初始化过程。

```java
// LogbackLoggingSystem.class

// 根据上面AbstractLoggingSystem的分析
// 使用logback日志库的时候会查找classpath下是否存在这些文件
// logback-test.groovy、logback-test.xml、logback.groovy、logback.xml以及
// logback-test-spring.groovy、logback-test-spring.xml、logback-spring.groovy、logback-spring.xml
@Override
protected String[] getStandardConfigLocations() {
	return new String[] { "logback-test.groovy", "logback-test.xml", "logback.groovy",
			"logback.xml" };
}

// logback具体的初始化加载过程
@Override
protected void loadConfiguration(LoggingInitializationContext initializationContext,
		String location, LogFile logFile) {
	// 调用父类Slf4JLoggingSystem的loadConfiguration方法
	super.loadConfiguration(initializationContext, location, logFile);
	// 获取slf4j内部的LoggerContext
	LoggerContext loggerContext = getLoggerContext();
	// logback环境的一些配置配置处理
	stopAndReset(loggerContext);
	try {
		configureByResourceUrl(initializationContext, loggerContext,
				ResourceUtils.getURL(location));
	}
	catch (Exception ex) {
		throw new IllegalStateException(
				"Could not initialize Logback logging from " + location, ex);
	}
	List<Status> statuses = loggerContext.getStatusManager().getCopyOfStatusList();
	StringBuilder errors = new StringBuilder();
	for (Status status : statuses) {
		if (status.getLevel() == Status.ERROR) {
			errors.append(errors.length() > 0 ? "\n" : "");
			errors.append(status.toString());
		}
	}
	if (errors.length() > 0) {
		throw new IllegalStateException(
				"Logback configuration error " + "detected: \n" + errors);
	}
}

// 没找到日志配置文件的话使用loadDefaults方法加载
@Override
protected void loadDefaults(LoggingInitializationContext initializationContext,
		LogFile logFile) {
	// 获取slf4j内部的LoggerContext
	LoggerContext context = getLoggerContext();
	stopAndReset(context);
	LogbackConfigurator configurator = new LogbackConfigurator(context);
	context.putProperty("LOG_LEVEL_PATTERN",
			initializationContext.getEnvironment().resolvePlaceholders(
					"${logging.pattern.level:${LOG_LEVEL_PATTERN:%5p}}"));
	// 构造默认的console Appender。如果logFile不为空，还会构造file Appender
	new DefaultLogbackConfiguration(initializationContext, logFile)
			.apply(configurator);
	context.setPackagingDataEnabled(true);
}

// logback的清除工作
@Override
public void cleanUp() {
	super.cleanUp();
	getLoggerContext().getStatusManager().clear();
}

// 动态设置logger的level
@Override
public void setLogLevel(String loggerName, LogLevel level) {
	getLogger(loggerName).setLevel(LEVELS.get(level));
}

// 清除后的一些工作
// ShutdownHandler会调用LoggerContext的stop方法
@Override
public Runnable getShutdownHandler() {
	return new ShutdownHandler();
}
```

## LoggingSystem的初始化

LoggingApplicationListener是ApplicationListener接口的实现类，会被springboot使用工厂加载机制加载：

```java
// spring-boot-version.jar/META-INF/spring.factories
# Application Listeners
org.springframework.context.ApplicationListener=\
org.springframework.boot.builder.ParentContextCloserApplicationListener,\
org.springframework.boot.context.FileEncodingApplicationListener,\
org.springframework.boot.context.config.AnsiOutputApplicationListener,\
org.springframework.boot.context.config.ConfigFileApplicationListener,\
org.springframework.boot.context.config.DelegatingApplicationListener,\
org.springframework.boot.liquibase.LiquibaseServiceLocatorApplicationListener,\
org.springframework.boot.logging.ClasspathLoggingApplicationListener,\
org.springframework.boot.logging.LoggingApplicationListener
```

```java
// SpringApplication.class
private void initialize(Object[] sources) {
	if (sources != null && sources.length > 0) {
		this.sources.addAll(Arrays.asList(sources));
	}
	this.webEnvironment = deduceWebEnvironment();
	setInitializers((Collection) getSpringFactoriesInstances(
			ApplicationContextInitializer.class));
	// 使用工厂加载机制找到这些Listener
	setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
	this.mainApplicationClass = deduceMainApplicationClass();
}

// LoggingApplicationListener.class
@Override
public void onApplicationEvent(ApplicationEvent event) {
	// SpringApplication的run方法执行的时候触发该事件
	if (event instanceof ApplicationStartedEvent) {
		// onApplicationStartedEvent方法内部会先得到LoggingSystem，然后调用beforeInitialize方法
		onApplicationStartedEvent((ApplicationStartedEvent) event);
	}
	// 环境信息准备好，ApplicationContext创建之前触发该事件
	else if (event instanceof ApplicationEnvironmentPreparedEvent) {
		// onApplicationEnvironmentPreparedEvent方法内部会做一下几个事情
		// 1. 读取配置文件中"logging."开头的配置，比如logging.pattern.level, logging.pattern.console等设置到系统属性中
		// 2. 构造一个LogFile(LogFile是对日志对外输出文件的封装)，使用LogFile的静态方法get构造，会使用配置文件中logging.file和logging.path配置构造
		// 3. 判断配置中是否配置了debug并为true，如果是，设置level的DEBUG，然后继续查看是否配置了trace并为true，如果是，设置level的TRACE
		// 4. 构造LoggingInitializationContext，查看是否配置了logging.config，如有配置，调用LoggingSystem的initialize方法并带上该参数，否则调用initialize方法并且configLocation为null
		// 5. 设置一些比如org.springframework.boot、org.springframework、org.apache.tomcat、org.apache.catalina、org.eclipse.jetty、org.hibernate.tool.hbm2ddl、org.hibernate.SQL这些包的log level，跟第3步的level一样
		// 6. 查看是否配置了logging.register-shutdown-hook，如配置并设置为true，使用addShutdownHook的addShutdownHook方法加入LoggingSystem的getShutdownHandler
		onApplicationEnvironmentPreparedEvent(
				(ApplicationEnvironmentPreparedEvent) event);
	}
	// Spring容器创建好，并进行了部分操作之后触发该事件
	else if (event instanceof ApplicationPreparedEvent) {
		// onApplicationPreparedEvent方法内部会把LoggingSystem注册到BeanFactory中(前期是BeanFactory中不存在name为springBootLoggingSystem的实例)
		onApplicationPreparedEvent((ApplicationPreparedEvent) event);
	}
	// Spring容器关闭的时候触发该事件
	else if (event instanceof ContextClosedEvent && ((ContextClosedEvent) event)
			.getApplicationContext().getParent() == null) {
		// onContextClosedEvent方法内部调用LoggingSystem的cleanUp方法进行清除工作
		onContextClosedEvent();
	}
}

private void onApplicationStartedEvent(ApplicationStartedEvent event) {
	// 一开始先使用LoggingSystem的静态方法get获取LoggingSystem
	// 静态方法get会从下面那段static代码块中得到的Map中进行遍历
	// 如果对应的key(key是某个类的全名)在classloader中存在，那么会构造该key对应的value对应的LoggingSystem
	this.loggingSystem = LoggingSystem
			.get(event.getSpringApplication().getClassLoader());
	this.loggingSystem.beforeInitialize();
}

static {
	Map<String, String> systems = new LinkedHashMap<String, String>();
	systems.put("ch.qos.logback.core.Appender",
			"org.springframework.boot.logging.logback.LogbackLoggingSystem");
	systems.put("org.apache.logging.log4j.core.impl.Log4jContextFactory",
			"org.springframework.boot.logging.log4j2.Log4J2LoggingSystem");
	systems.put("org.apache.log4j.PropertyConfigurator",
			"org.springframework.boot.logging.log4j.Log4JLoggingSystem");
	systems.put("java.util.logging.LogManager",
			"org.springframework.boot.logging.java.JavaLoggingSystem");
	SYSTEMS = Collections.unmodifiableMap(systems);
}

private void onApplicationEnvironmentPreparedEvent(
		ApplicationEnvironmentPreparedEvent event) {
	if (this.loggingSystem == null) {
		this.loggingSystem = LoggingSystem
				.get(event.getSpringApplication().getClassLoader());
	}
	initialize(event.getEnvironment(), event.getSpringApplication().getClassLoader());
}

private void onApplicationPreparedEvent(ApplicationPreparedEvent event) {
	ConfigurableListableBeanFactory beanFactory = event.getApplicationContext()
			.getBeanFactory();
	if (!beanFactory.containsBean(LOGGING_SYSTEM_BEAN_NAME)) {
		beanFactory.registerSingleton(LOGGING_SYSTEM_BEAN_NAME, this.loggingSystem);
	}
}

private void onContextClosedEvent() {
	if (this.loggingSystem != null) {
		this.loggingSystem.cleanUp();
	}
}

```

spring-boot-starter模块内部会引用spring-boot-starter-logging模块，这个starter-logging模块内部会引入logback相关的依赖。这一依赖会导致LoggingSystem的静态方法get获取LoggingSystem的时候会得到LogbackLoggingSystem。

因此默认情况下，springboot程序基本都是使用logback作为默认的日志。

## 一些例子

前提都是以LogbackLoggingSystem作为日志系统。

### 项目里没有任何日志的配置

由于没有任何配置，进行initialize的时候日志配置文件为null，最后只能调用loadDefaults方法进行加载，LogbackLoggingSystem的loadDefaults方法，由于logFile为null，所以最终只构造了一个ConsoleAppender。

所以项目没有任何日志配置的情况下，控制台依旧能打印出项目的启动信息。


### 项目里没有任何logback的配置，只有yaml中配置了logging.file和logging.path

logging.file和logging.path的配置在LogFile这个日志文件类中生效。

比如yaml配置如下(只定义了logging.file)：

```yaml
logging:
  file: /tmp/temp.log
```

这个配置导致了调用initialize方法的时候logFile存在，这样不止有ConsoleAppender，还有一个FileAppender，这个FileAppender对应的文件就是LogFile文件，也就是 /tmp/temp.log日志文件。

比如yaml配置如下(只定义了logging.path)：

```yaml
logging:
  path: /tmp
```

这个时候FileAppender对应的file是/tmp/spring.log文件。

```java
// LogFile.class
@Override
public String toString() {
	// 如果配置了logging.file，直接使用该文件
	if (StringUtils.hasLength(this.file)) {
		return this.file;
	}
	// 否则使用logging.path目录，在该目录下创建spring.log日志文件
	String path = this.path;
	if (!path.endsWith("/")) {
		path = path + "/";
	}
	return StringUtils.applyRelativePath(path, "spring.log");
}
```

所以我们如果配置了logging.path和logging.file，那么生效的只有logging.file配置。

### resources下有logback.xml配置(相当于就是classpath下存在logback.xml文件)

之前分析过，LogbackLoggingSystem中的getStandardConfigLocations方法返回以下文件：

1. logback-test.groovy或者logback-test-spring.groovy
2. logback-test.xml或者logback-test-spring.xml
3. logback.groovy或者logback-spring.groovy
4. logback.xml或者logback-spring.xml

在resources目录下定义logback-spring.xml文件，内容如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <!--<pattern>%d{YYYY-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger[%line] &#45;&#45; %msg%n</pattern>-->
            <pattern>%d{YYYY-MM-dd} [%thread] %-5level %logger[%line] -- %msg%n</pattern>
        </encoder>
    </appender>

    <root level="INFO">
        <appender-ref ref="STDOUT"/>
    </root>
</configuration>
```

这时logging.file配置失效，这是因为没有调用loadDefaults方法(loadDefaults方法内部会把LogFile构造成FileAppender)，而是调用了loadConfiguration方法，该方法会根据logback.xml文件中的配置去构造Appender。

### resources下有my-logback.xml配置

由于LogbackLoggingSystem中没有对my-logback.xml路径的解析，所有不会被识别，但是可以在yaml中配置logging.config配置：

```yaml
logging:
  config: classpath:my-logback.xml
```

这样配置就能识别my-logback.xml文件。

## 其它

最新版的SpringBoot内部提供了一个NoOpLoggingSystem，这个日志系统内部什么都不做，构造过程如下：

```java
public static LoggingSystem get(ClassLoader classLoader) {
	// SYSTEM_PROPERTY静态变量是LoggingSystem的类全名
	String loggingSystem = System.getProperty(SYSTEM_PROPERTY);
	if (StringUtils.hasLength(loggingSystem)) {
		if (NONE.equals(loggingSystem)) { // None静态变量是值是none
			return new NoOpLoggingSystem();
		}
		return get(classLoader, loggingSystem);
	}
	for (Map.Entry<String, String> entry : SYSTEMS.entrySet()) {
		if (ClassUtils.isPresent(entry.getKey(), classLoader)) {
			return get(classLoader, entry.getValue());
		}
	}
	throw new IllegalStateException("No suitable logging system located");
}
```

所以程序启动的时候加上 -Dorg.springframework.boot.logging.LoggingSystem=none就可以构造NoOpLoggingSystem这个日志系统。
