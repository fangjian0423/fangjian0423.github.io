title: SpringBoot编写自定义的starter
date: 2016-11-16 01:40:32
tags:
- springboot
categories: springboot

----------------

在之前的文章中，我们分析过SpringBoot内部的[自动化配置原理](http://fangjian0423.github.io/2016/06/12/springboot-autoconfig-analysis/)和[自动化配置注解开关原理](http://fangjian0423.github.io/2016/11/13/springboot-enable-annotation/)。

我们先简单分析一下[mybatis starter](https://github.com/mybatis/spring-boot-starter)的编写，然后再编写自定义的starter。

mybatis中的autoconfigure模块中使用了一个叫做MybatisAutoConfiguration的自动化配置类。

这个MybatisAutoConfiguration需要在这些Condition条件下才会执行：

1. @ConditionalOnClass({ SqlSessionFactory.class, SqlSessionFactoryBean.class })。需要SqlSessionFactory和SqlSessionFactoryBean在classpath中都存在
2. @ConditionalOnBean(DataSource.class)。 spring factory中需要存在一个DataSource的bean
3. @AutoConfigureAfter(DataSourceAutoConfiguration.class)。需要在DataSourceAutoConfiguration自动化配置之后进行配置，因为mybatis需要数据源的支持


同时在META-INF目录下有个spring.factories这个properties文件，而且它的key为org.springframework.boot.autoconfigure.EnableAutoConfiguration，这样才会被springboot加载：

	# Auto Configure
	org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
	org.mybatis.spring.boot.autoconfigure.MybatisAutoConfiguration


有了这些东西之后，mybatis相关的配置会被自动加入到spring container中，只要在maven中加入starter即可：

	<dependency>
    	<groupId>org.mybatis.spring.boot</groupId>
	    <artifactId>mybatis-spring-boot-starter</artifactId>
    	<version>1.1.1</version>
	</dependency>

<!--more-->

## 编写自定义的starter

接下来，我们来编写自定义的starter：log-starter。

这个starter内部定义了一个注解，使用这个注解修饰方法之后，该方法的调用会在日志中被打印并且还会打印出方法的耗时。starter支持exclude配置，在exclude中出现的方法不会进行计算。

pom文件：

	<parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starters</artifactId>
        <version>1.3.5.RELEASE</version>
    </parent>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>
    </dependencies>

定义修饰方法的注解@Log：

	package me.format.springboot.log.annotation;

	import java.lang.annotation.ElementType;
	import java.lang.annotation.Retention;
	import java.lang.annotation.RetentionPolicy;
	import java.lang.annotation.Target;	

	@Retention(RetentionPolicy.RUNTIME)
	@Target(ElementType.METHOD)
	public @interface Log { }

然后是配置类：

	package me.format.springboot.log.autoconfigure;
    
    import org.springframework.boot.context.properties.ConfigurationProperties;
    import org.springframework.util.StringUtils;
    
    import javax.annotation.PostConstruct;
    
    @ConfigurationProperties(prefix = "mylog")
    public class LogProperties {
    
        private String exclude;
    
        private String[] excludeArr;
    
        @PostConstruct
        public void init() {
            this.excludeArr = StringUtils.split(exclude, ",");
        }
    
        public String getExclude() {
            return exclude;
        }
    
        public void setExclude(String exclude) {
            this.exclude = exclude;
        }
    
        public String[] getExcludeArr() {
            return excludeArr;
        }
    }


接下来是AutoConfiguration：

	package me.format.springboot.log.autoconfigure;
    
    import me.format.springboot.log.annotation.Log;
    import me.format.springboot.log.aop.LogMethodInterceptor;
    import org.aopalliance.aop.Advice;
    import org.slf4j.Logger;
    import org.slf4j.LoggerFactory;
    import org.springframework.aop.Pointcut;
    import org.springframework.aop.support.AbstractPointcutAdvisor;
    import org.springframework.aop.support.annotation.AnnotationMatchingPointcut;
    import org.springframework.beans.factory.annotation.Autowired;
    import org.springframework.boot.context.properties.EnableConfigurationProperties;
    import org.springframework.context.annotation.Configuration;
    
    import javax.annotation.PostConstruct;
    
    @Configuration
    @EnableConfigurationProperties(LogProperties.class)
    public class LogAutoConfiguration extends AbstractPointcutAdvisor {
    
        private Logger logger = LoggerFactory.getLogger(LogAutoConfiguration.class);
    
        private Pointcut pointcut;
    
        private Advice advice;
    
        @Autowired
        private LogProperties logProperties;
    
        @PostConstruct
        public void init() {
            logger.info("init LogAutoConfiguration start");
            this.pointcut = new AnnotationMatchingPointcut(null, Log.class);
            this.advice = new LogMethodInterceptor(logProperties.getExcludeArr());
            logger.info("init LogAutoConfiguration end");
        }
    
        @Override
        public Pointcut getPointcut() {
            return this.pointcut;
        }
    
        @Override
        public Advice getAdvice() {
            return this.advice;
        }
    
    }


由于计算方法调用的时候需要使用aop相关的lib，所以我们的AutoConfiguration继承了AbstractPointcutAdvisor。这样就有了Pointcut和Advice。Pointcut是一个支持注解的修饰方法的Pointcut，Advice则自己实现：

	package me.format.springboot.log.aop;
    
    import org.aopalliance.intercept.MethodInterceptor;
    import org.aopalliance.intercept.MethodInvocation;
    import org.slf4j.Logger;
    import org.slf4j.LoggerFactory;
    
    import java.util.Arrays;
    import java.util.List;
    
    public class LogMethodInterceptor implements MethodInterceptor {
        private Logger logger = LoggerFactory.getLogger(LogMethodInterceptor.class);
        private List<String> exclude;
        public LogMethodInterceptor(String[] exclude) {
            this.exclude = Arrays.asList(exclude);
        }
        @Override
        public Object invoke(MethodInvocation invocation) throws Throwable {
            String methodName = invocation.getMethod().getName();
            if(exclude.contains(methodName)) {
                return invocation.proceed();
            }
            long start = System.currentTimeMillis();
            Object result = invocation.proceed();
            long end = System.currentTimeMillis();
            logger.info("====method({}), cost({}) ", methodName, (end - start));
            return result;
        }
    }


最后resources/META-INF/spring.factories中加入这个AutoConfiguration：

	org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
	me.format.springboot.log.autoconfigure.LogAutoConfiguration


我们在项目中使用这个log-starter：

	<dependency>
        <groupId>me.format.springboot</groupId>
        <artifactId>log-starter</artifactId>
        <version>1.0-SNAPSHOT</version>
    </dependency>

使用配置：

	mylog.exclude=core,log

然后编写一个简单的Service：

	@Service
	public class SimpleService {

    	@Log
	    public void test(int num) {
    	    System.out.println("----test---- " + num);
	    }

    	@Log
	    public void core(int num) {
        	System.out.println("----core---- " + num);
    	}

	    public void work(int num) {
        	System.out.println("----work---- " + num);
    	}
	
	}


使用单元测试分别调用这3个方法，由于work方法没有加上@Log注解，core方法虽然加上了@Log注解，但是在配置中被exclude了，只有test方法可以正常计算耗时：

	----test---- 666
	2016-11-16 01:29:32.255  INFO 41010 --- [           main] m.f.s.log.aop.LogMethodInterceptor       : ====method(test), 	cost(36) 
	----work---- 666
	----core---- 666
	
总结：

自定义springboot的starter，注意这两点。

1. 如果自动化配置类需要在程序启动的时候就加载，可以在META-INF/spring.factories文件中定义。如果本次加载还需要其他一些lib的话，可以使用ConditionalOnClass注解协助
2. 如果自动化配置类要在使用自定义注解后才加载，可以使用自定义注解+@Import注解或@ImportSelector注解完成

参考：

http://www.jianshu.com/p/85460c1d835a

http://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-developing-auto-configuration.html


