title: SpringBoot的actuator模块
date: 2016-06-25 14:52:52
tags:
- springboot
- scala
categories: springboot

----------------

springboot内部提供了一个模块spring-boot-actuator用于监控和管理springboot应用。

这个模块内部提供了很多功能，endpoint就是其中一块功能。

我们在sbt中加入这个模块的依赖：

	libraryDependencies += "org.springframework.boot" % "spring-boot-starter-actuator" % "1.3.5.RELEASE"
	
然后启动项目，访问地址 http://localhost:8080/health，看到以下页面：

	{
		status: "UP",
		diskSpace: {
			status: "UP",
			total: 249779191808,
			free: 22231089152,
			threshold: 10485760
		},
		db: {
			status: "UP",
			database: "H2",
			hello: 1
		}
	}
	
<!--more-->
	
这个/health endpoint显示了目前应用的一些健康情况。

## Endpoint

spring-boot-autoconfigure模块中还提供了其他很多endpoint，比如 /beans(查看spring工厂信息，里面存在哪些bean)、/dump(应用中的所有线程状态)、/env(应用环境信息，比如jvm环境信息、配置文件内容、应用端口等信息)、/mappings(SpringMVC的RequestMapping映射信息)、/configprops(框架配置信息，比如数据源、freemarker、spring的配置信息)、/metrics(度量信息)等等。

具体其他的endpoint可以查看springboot官方文档上的信息： http://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#production-ready-endpoints

这些endpoint是如何暴露出来的呢，是通过SpringBoot内部的一个Endpoint接口完成的。

	public interface Endpoint<T> {
		String getId(); // endpoint的唯一标识
		boolean isEnabled(); // 是否可用
		boolean isSensitive(); // 是否对一般用户可见
		T invoke(); // 具体的执行过程，返回值会被解析成json暴露出口
	}
	
这个接口的实现类有DumpEndpoint、BeansEndpoint、InfoEndpoint、HealthEndpoint、RequestMappingEndpoint等等。这些EndPoint实现类就是对应对外暴露的endpoint。BeansEndpoint的代码如下：

	@ConfigurationProperties(prefix = "endpoints.beans") // 配置以endpoints.beans开头，可以覆盖
	public class BeansEndpoint extends AbstractEndpoint<List<Object>>
			implements ApplicationContextAware {

		private final LiveBeansView liveBeansView = new LiveBeansView();

		private final JsonParser parser = JsonParserFactory.getJsonParser();

		public BeansEndpoint() {
			super("beans"); // id为beans
		}

		@Override
		public void setApplicationContext(ApplicationContext context) throws BeansException {
			if (context.getEnvironment()
					.getProperty(LiveBeansView.MBEAN_DOMAIN_PROPERTY_NAME) == null) {
				this.liveBeansView.setApplicationContext(context);
			}
		}

		@Override
		public List<Object> invoke() {
			return this.parser.parseList(this.liveBeansView.getSnapshotAsJson()); // 返回值就是最后展示的json数组
		}
	}

可以使用配置覆盖默认的beans endpoint中的信息：

	endpoints.beans.enabled= # Enable the endpoint.
	endpoints.beans.id= # Endpoint identifier.
	endpoints.beans.path= # Endpoint path.
	endpoints.beans.sensitive= # Mark if the endpoint exposes sensitive information.

## Health Indicator

HealthEndpoint这个endpoint是暴露通过扫描出的HealthIndicator接口的实现类完成的。

用于查看应用的健康状况。

在我们的应用中只使用了h2数据库，最终/health显示出来的内容如下(只有硬盘容量和数据库健康状况)：

	{
		status: "UP",
		diskSpace: {
			status: "UP",
			total: 249779191808,
			free: 20595720192,
			threshold: 10485760
		},
		db: {
			status: "UP",
			database: "H2",
			hello: 1
		}
	}
	
springboot内置的HealthIndicator有这些：SolrHealthIndicator、RedisHealthIndicator、RabbitHealthIndicator、MongoHealthIndicator、ElasticsearchHealthIndicator、CassandraHealthIndicator、DiskSpaceHealthIndicator、DataSourceHealthIndicator等。

这些都是springboot内置的，我们也可以编写自定义的health indicator。

以服务器中某个目录中的文件个数不能超过某个值为健康指标作为需求进行编写health indicator。

编写HealthIndicator：

	@Component
	public class TempDirHealthIndicator implements HealthIndicator {

	    @Override
	    public Health health() {
	        Health.Builder builder = new Health.Builder();
	        File file = new File("youdirpath");
	        File[] fileList = file.listFiles();
	        if(fileList.length > 10) {
	            builder.down().withDetail("file num",fileList.length);
	        } else {
	            builder.up();
	        }
	        return builder.build();
	    }

	}

	
访问 http://localhost:8080/health

	{
		status: "DOWN",
		tempDir: {
			status: "DOWN",
			file num: 34
		},
		diskSpace: {
			status: "UP",
			total: 249779191808,
			free: 20690649088,
			threshold: 10485760
		},
		db: {
			status: "UP",
			database: "H2",
			hello: 1
		}
	}


## Metrics


Metrics服务用来做一些度量支持，springboot提供了两种Metrics，分别是gauge(单一的值)和counter(计数器，自增或自减)。springboot提供了PublicMetrics接口用来支持Metrics服务。

metrics这个endpoint中使用的metrics都是由SystemPublicMetrics完成的：

	{
		mem: 388503,
		mem.free: 199992,
		processors: 4,
		instance.uptime: 58260089,
		uptime: 21843805,
		systemload.average: 3.28369140625,
		heap.committed: 320000,
		heap.init: 131072,
		heap.used: 120007,
		heap: 1864192,
		nonheap.committed: 69528,
		nonheap.init: 2496,
		nonheap.used: 68501,
		nonheap: 0,
		threads.peak: 15,
		threads.daemon: 13,
		threads.totalStarted: 20,
		threads: 15,
		classes: 9483,
		classes.loaded: 9484,
		classes.unloaded: 1,
		gc.ps_scavenge.count: 9,
		gc.ps_scavenge.time: 152,
		gc.ps_marksweep.count: 2,
		gc.ps_marksweep.time: 167,
		httpsessions.max: -1,
		httpsessions.active: 0,
		datasource.primary.active: 0,
		datasource.primary.usage: 0,
		gauge.response.health: 218,
		gauge.response.star-star.favicon.ico: 29,
		counter.status.200.star-star.favicon.ico: 1,
		counter.status.503.health: 1
	}

gauge和counter度量通过GaugeService和CounterService完成。

比如我们要查看各个Controller中的接口被调用的次数话，可以使用CounterService和aop完成：


	@Aspect
	@Component
	class ControllerAspect @Autowired() (
	  counterService: CounterService
	) {

	  @Before("execution(* me.format.controller.*.*(..))")
	  def controllerCounter(joinPoint: JoinPoint): Unit = {
	    counterService.increment(joinPoint.getSignature.toString + "-invokeNum")
	  }

	}

启动应用，调用getPersons接口4次，调用get/{long} 2次，查看metrics endpoint，得到以下信息：

	counter.List me.format.controller.PersonController.get(HttpServletRequest)-invokeNum: 4,
	counter.Person me.format.controller.PersonController.get(long)-invokeNum: 2,
	
	
	
比如我们要查看各个Controller中的接口的延迟情况，可以使用GaugeService和aop完成：

	@Aspect
	@Component
	class ControllerAspect @Autowired() (
	  counterService: CounterService,
	  gaugeService: GaugeService
	) {

	  @Before("execution(* me.format.controller.*.*(..))")
	  def controllerCounter(joinPoint: JoinPoint): Unit = {
	    counterService.increment(joinPoint.getSignature.toString + "-invokeNum")
	  }

	  @Around("execution(* me.format.controller.*.*(..))")
	  def controllerGauge(proceedingJoinPoint: ProceedingJoinPoint): AnyRef = {
	    val st = System.currentTimeMillis()
	    val result = proceedingJoinPoint.proceed()
	    val et = System.currentTimeMillis()
	    gaugeService.submit(proceedingJoinPoint.getSignature.toString + "-invokeTime", (et - st))
	    result
	  }

	}
	
启动应用，调用getPersons接口4次，调用get/{long} 2次，查看metrics endpoint，得到以下信息：

	gauge.List me.format.controller.PersonController.get(HttpServletRequest)-invokeTime: 4,
	gauge.Person me.format.controller.PersonController.get(long)-invokeTime: 11,
	counter.List me.format.controller.PersonController.get(HttpServletRequest)-invokeNum: 4,
	counter.Person me.format.controller.PersonController.get(long)-invokeNum: 2,
	
当然，actuator模块提供的功能远不止这些，更多的信息可以查看官方文档。

http://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#production-ready
