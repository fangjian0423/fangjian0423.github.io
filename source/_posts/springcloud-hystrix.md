title: SpringCloud使用Hystrix实现断路器
date: 2017-02-19 12:11:39
tags:
- springboot
- springcloud
categories: springcloud

----------------

[Hystrix](https://github.com/Netflix/Hystrix)是一个供分布式系统使用，提供延迟和容错功能，保证复杂的分布系统在面临不可避免的失败时，仍能有其弹性。

比如系统中有很多服务，当某些服务不稳定的时候，使用这些服务的用户线程将会阻塞，如果没有隔离机制，系统随时就有可能会挂掉，从而带来很大的风险。

SpringCloud使用Hystrix组件提供断路器、资源隔离与自我修复功能。下图表示服务B触发了断路器，阻止了级联失败。

![image](http://cloud.spring.io/spring-cloud-static/Brixton.SR7/images/HystrixFallback.png)


<!--more-->

## Hystrix的简单使用

Hystrix使用了命令设计模式，只需要编写命令即可：

	public class CommandHelloWorld extends HystrixCommand<String> {

        private final String name;

        public CommandHelloWorld(String name) {
            super(HystrixCommandGroupKey.Factory.asKey("HelloWorld"));
            this.name = name;
        }

        @Override
        protected String run() throws Exception { // 完成业务逻辑
            return "Hello " + name + "!";
        }

        @Override
	    protected String getFallback() { // run方法抛出异常的时候返回备用结果
    	    return "Hello Failure " + name + "!";
	    }

    }

测试用例：

	@Test
    public void test() {
        assertEquals("Hello World!", new CommandHelloWorld("World").execute());
        assertEquals("Hello Format!", new CommandHelloWorld("Format").execute());
    }


可能有的人觉得写Command有点麻烦，Hystrix提供了一个类库[javanica](https://github.com/Netflix/Hystrix/tree/master/hystrix-contrib/hystrix-javanica)，可以使用@HystrixCommand注解完成命令的编写。


## 在SpringCloud中使用Hystrix

要在SpringCloud中使用断路器，需要加上@EnableCircuitBreaker注解：

	...
    @EnableCircuitBreaker
    ...
	public class RibbonApplication { ... }


然后在对应的方法上加入@HystrixCommand注解实现断路器功能，当service方法对应的服务发生异常的时候，会跳转到serviceFallback方法执行：

	@HystrixCommand(fallbackMethod = "serviceFallback") // 加入@HystrixCommand注解实现断路器功能
    public String service() { // 原先的方法
        return restTemplate.getForEntity("...", String.class).getBody();
    }

    public String serviceFallback() { // fallback方法
        return "error";
    }

## 工作原理

加上@EnableCircuitBreaker注解之后，就可以使用断路器功能，所以SpringCloud内部是如何整合Hystrix的话先从这个注解开始分析。

@EnableCircuitBreaker注解定义如下：

	@Target(ElementType.TYPE)
	@Retention(RetentionPolicy.RUNTIME)
	@Documented
	@Inherited
	@Import(EnableCircuitBreakerImportSelector.class)
	public @interface EnableCircuitBreaker {

	}

import了EnableCircuitBreakerImportSelector这个selector：

	public class EnableCircuitBreakerImportSelector extends
		SpringFactoryImportSelector<EnableCircuitBreaker> {

		@Override
		protected boolean isEnabled() {
			return new RelaxedPropertyResolver(getEnvironment()).getProperty(
				"spring.cloud.circuit.breaker.enabled", Boolean.class, Boolean.TRUE);
		}

	}

在之前的这篇[SpringBoot自动化配置的注解开关原理](http://fangjian0423.github.io/2016/11/13/springboot-enable-annotation/)文章中分析过selector的原理，这个EnableCircuitBreakerImportSelector会加载spring.factories属性文件中key为org.springframework.cloud.client.circuitbreaker.EnableCircuitBreaker的类：

	org.springframework.cloud.client.circuitbreaker.EnableCircuitBreaker=\
		org.springframework.cloud.netflix.hystrix.HystrixCircuitBreakerConfiguration

会加载HystrixCircuitBreakerConfiguration这个配置类。

这个配置类内部构造了一个aspect：

	@Bean
	public HystrixCommandAspect hystrixCommandAspect() {
		return new HystrixCommandAspect();
	}

这个aspect对应的pointcut如下，所以使用@HystrixCommand注解修饰的方法会被这个aspect处理：

	@annotation(com.netflix.hystrix.contrib.javanica.annotation.HystrixCommand) || @annotation(com.netflix.hystrix.contrib.javanica.annotation.HystrixCollapser)

对应的aop处理方法：

	public Object methodsAnnotatedWithHystrixCommand(final ProceedingJoinPoint joinPoint) throws Throwable {
        Method method = getMethodFromTarget(joinPoint);  // 得到初始的方法
        Validate.notNull(method, "failed to get method from joinPoint: %s", joinPoint);
        if (method.isAnnotationPresent(HystrixCommand.class) && method.isAnnotationPresent(HystrixCollapser.class)) { // 如果使用@HystrixCommand注解和@HystrixCollapser注解同时修改，不允许
            throw new IllegalStateException("method cannot be annotated with HystrixCommand and HystrixCollapser " +
                    "annotations at the same time");
        }
        MetaHolderFactory metaHolderFactory = META_HOLDER_FACTORY_MAP.get(HystrixPointcutType.of(method));
        MetaHolder metaHolder = metaHolderFactory.create(joinPoint); // 创建一个MetaHolder，这个MetaHolder封装了方法中的一些以及Hystrix的一些信息
        HystrixInvokable invokable = HystrixCommandFactory.getInstance().create(metaHolder); // 根据这个metaHolder创建出一个HystrixInvokable，也就是一个HystrixCommand
        ExecutionType executionType = metaHolder.isCollapserAnnotationPresent() ?
                metaHolder.getCollapserExecutionType() : metaHolder.getExecutionType(); // 得到执行类型，有3种类型：1. 异步 2. 同步  3. reactive
        Object result;
        try {
            result = CommandExecutor.execute(invokable, executionType, metaHolder);
        } catch (HystrixBadRequestException e) {
            throw e.getCause();
        }
        return result;
    }

CommandExecutor的execute方法：

	public static Object execute(HystrixInvokable invokable, ExecutionType executionType, MetaHolder metaHolder) throws RuntimeException {
        Validate.notNull(invokable);
        Validate.notNull(metaHolder);

        switch (executionType) {
            case SYNCHRONOUS: { // 同步方式的话，调用HystrixCommand的execute方法
                return castToExecutable(invokable, executionType).execute();
            }
            case ASYNCHRONOUS: { // 异步方式的话，调用HystrixCommand的queue方法
                HystrixExecutable executable = castToExecutable(invokable, executionType);
                if (metaHolder.hasFallbackMethodCommand()
                        && ExecutionType.ASYNCHRONOUS == metaHolder.getFallbackExecutionType()) {
                    return new FutureDecorator(executable.queue());
                }
                return executable.queue();
            }
            case OBSERVABLE: { // reactive方式的话，调用HystrixCommand的observe或者toObservable方法
                HystrixObservable observable = castToObservable(invokable);
                return ObservableExecutionMode.EAGER == metaHolder.getObservableExecutionMode() ? observable.observe() : observable.toObservable();
            }
            default:
                throw new RuntimeException("unsupported execution type: " + executionType);
        }
    }

根据metaHolder创建出HystrixCommand的过程在HystrixCommandBuilderFactory中：

	return HystrixCommandBuilder.builder()
                .setterBuilder(createGenericSetterBuilder(metaHolder))
                .commandActions(createCommandActions(metaHolder))
                .collapsedRequests(collapsedRequests)
                .cacheResultInvocationContext(createCacheResultInvocationContext(metaHolder))
                .cacheRemoveInvocationContext(createCacheRemoveInvocationContext(metaHolder))
                .ignoreExceptions(metaHolder.getHystrixCommand().ignoreExceptions())
                .executionType(metaHolder.getExecutionType())
                .build();


所以这个aspect的作用就是把一个普通的Java方法转换成HystrixCommand。

## 其它

HystrixCircuitBreakerConfiguration配置类中有个HystrixWebConfiguration内部配置类，它构造了一个HystrixStreamEndpoint这个endpoint，这个endpoint使用HystrixMetricsStreamServlet暴露出/hystrix.stream地址来获取hystrix的metrics信息。

Hystrix还提供了一个dashboard，这个dashboard可以查看各个断路器的健康状况，要使用这个dashboard，在项目中加入这些依赖：

	<dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-hystrix-dashboard</artifactId>
    </dependency>

然后在代码里加上开关：

	@EnableHystrixDashboard
	...

启动项目，打开：

	http://localhost:3333/hystrix


![image](http://7x2wh6.com1.z0.glb.clouddn.com/hystrix01.png)
输入：

	http://localhost:3333/hystrix.stream

我们使用wrk模拟请求：

	wrk -c 10 -t 10 -d 20s http://localhost:3333/add

然后dashboard中发生了变化：

![image](http://7x2wh6.com1.z0.glb.clouddn.com/hystrix02.png)


## 参考资料

http://cloud.spring.io/spring-cloud-static/Brixton.SR7/

https://github.com/Netflix/Hystrix

https://github.com/Netflix/Hystrix/wiki
