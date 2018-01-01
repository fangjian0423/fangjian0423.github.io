title: SpringBoot应用程序的关闭
date: 2017-06-28 09:15:30
tags:
- spring
- java
- springboot
categories: springboot

----------------

SpringBoot应用程序的关闭目前总结起来有4种方式：

1. Rest接口：使用spring-boot-starter-actuator模块里的ShutdownEndpoint
2. SpringApplication的exit静态方法：直接调用该静态方法即可
3. JMX：使用SpringBoot内部提供的MXBean
4. 使用第三方进程管理工具

<!--more-->


## Rest接口

Rest接口使用Endpoint暴露出来，需要引入[spring-boot-starter-actuator](https://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#production-ready-endpoints)这个stater。

这个关闭应用程序对应的Endpoint是ShutdownEndpoint，直接调用ShutdownEndpoint提供的rest接口即可。得先开启ShutdownEndpoint(默认不开启)，以及不进行安全监测：

```
endpoints.shutdown.enabled: true
endpoints.shutdown.sensitive: false
```

然后调用rest接口：

```shell
curl -X POST http://localhost:8080/shutdown
```

可以使用spring-security进行安全监测：

```
endpoints.shutdown.sensitive: true
security.user.name: admin
security.user.password: admin
management.security.role: SUPERUSER
```

然后使用用户名和密码进行调用：

```shell
curl -u admin:admin -X POST http://127.0.0.1:8080/shutdown
```

这个ShutdownEndpoint底层其实就是调用了Spring容器的close方法：

```java
@Override
public Map<String, Object> invoke() {

	if (this.context == null) {
		return Collections.<String, Object>singletonMap("message",
				"No context to shutdown.");
	}

	try {
		return Collections.<String, Object>singletonMap("message",
				"Shutting down, bye...");
	}
	finally {

		new Thread(new Runnable() {
			@Override
			public void run() {
				try {
					Thread.sleep(500L);
				}
				catch (InterruptedException ex) {
					// Swallow exception and continue
				}
				ShutdownEndpoint.this.context.close();
			}
		}).start();

	}
}
```

## SpringApplication的exit静态方法

SpringApplication提供了一个exit静态方法，用于关闭Spring容器，该方法还有一个参数exitCodeGenerators表示ExitCodeGenerator接口的数组。ExitCodeGenerator接口是一个生成退出码exitCode的生成器。

```java
public static int exit(ApplicationContext context,
		ExitCodeGenerator... exitCodeGenerators) {
	Assert.notNull(context, "Context must not be null");
	int exitCode = 0; // 默认的退出码是0
	try {
		try {
			// 构造ExitCodeGenerator集合
			ExitCodeGenerators generators = new ExitCodeGenerators();
			// 获得Spring容器中所有的ExitCodeGenerator类型的bean
			Collection<ExitCodeGenerator> beans = context
					.getBeansOfType(ExitCodeGenerator.class).values();
			// 集合加上参数中的ExitCodeGenerator数组
			generators.addAll(exitCodeGenerators);
			// 集合加上Spring容器中的ExitCodeGenerator集合
			generators.addAll(beans);
			// 遍历每个ExitCodeGenerator，得到最终的退出码exitCode
			// 这里每个ExitCodeGenerator生成的退出码如果比0大，那么取最大的
			// 如果比0小，那么取最小的
			exitCode = generators.getExitCode();
			if (exitCode != 0) { // 如果退出码exitCode不为0，发布ExitCodeEvent事件
				context.publishEvent(new ExitCodeEvent(context, exitCode));
			}
		}
		finally {
			// 关闭Spring容器
			close(context);
		}

	}
	catch (Exception ex) {
		ex.printStackTrace();
		exitCode = (exitCode == 0 ? 1 : exitCode);
	}
	return exitCode;
}
```

我们写个Controller直接调用exit方法：

```java
@Autowired
private ApplicationContext applicationContext;

@PostMapping("/stop")
public String stop() {
		// 加上自己的权限验证
    SpringApplication.exit(applicationContext);
    return "ok";
}
```


## JMX

[spring-boot-starter-actuator](https://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#production-ready-endpoints)这个stater内部会构造ShutdownEndpointMBean。

使用jconsole可以看到这个MBean：

![](http://7x2wh6.com1.z0.glb.clouddn.com/springboot-stop01.png)

SpringBoot内部也提供了一个SpringApplicationAdminMXBean，但是需要开启：

```
spring.application.admin.enabled: true
```

![](http://7x2wh6.com1.z0.glb.clouddn.com/springboot-stop02.png)


## 使用第三方进程管理工具

比如我们的应用程序部署在linux系统上，可以借助一些第三方的进程管理工具管理应用程序的运行，比如[supervisor](http://www.supervisord.org/)。

设置program：

```shell
[program:stop-application]
command=java -jar /yourjar.jar
process_name=%(program_name)s
startsecs=10
autostart=false
autorestart=false
stdout_logfile=/tmp/stop.log
stderr_logfile=/tmp/stop-error.log
```

使用supervisorctl进入控制台操作应用程序：


```shell
supervisor> status
stop-application                 STOPPED   Jun 27 03:50 PM
supervisor> start stop-application
stop-application: started
supervisor> status
stop-application                 RUNNING   pid 27918, uptime 0:00:11
supervisor> stop stop-application
stop-application: stopped
supervisor> status
stop-application                 STOPPED   Jun 27 03:50 PM
```
