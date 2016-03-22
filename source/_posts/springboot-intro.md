title: Spring Boot 部分特性记录
date: 2015-09-21 00:21:49
tags:
- springboot
- microservice
categories: 
- springboot
description: SpringBoot是Java的一个micro-service框架。它设计的目的是简化Spring应用的初始搭建以及开发过程。使用SpringBoot可以避免大量的xml配置文件，它内部使用很多约定的方式 ...

---------------

SpringBoot是Java的一个micro-service框架。它设计的目的是简化Spring应用的初始搭建以及开发过程。使用SpringBoot可以避免大量的xml配置文件，它内部使用很多约定的方式。

以一个最简单的MVC例子来说，使用SpringBoot进行开发的话定义好对应的Controller，Repository和Entity之后，加上各自的Annotation即可。

Repository框架可以选择Spring Data或者Hibernate，可通过自由配置。

视图框架也可通过配置选择freemarker或者velocity等视图框架。

下面，介绍一下SpringBoot的一些功能。


## SpringBoot框架的启动 ##

SpringBoot使用SpringApplication这个类进行项目的启动。一般都会这么写：

	@SpringBootApplication
    public class Application {
        public static void main(String[] args) {
            SpringApplication.run(Application.class, args);
        }
    }
    
    
使用SpringBootApplication注解相当于使用了3个注解，分别是@ComponentScan，@Configuration，@EnableAutoConfiguration。

这里需要注意的是Application这个类所有的package位置。

比如你的项目的包路径是 me.format.project1，对应的controller和repository包是 me.format.project1.controller和me.format.project1.repository。 那么这个Application需要在的包路径为 me.format.project1。 因为SpringBootApplication注解内部是使用ComponentScan注解，这个注解会扫描Application包所在的路径下的各个bean。


## Profile的使用 ##

可以在springboot项目中加入配置文件application.yml。

yaml中可以定义多个profile，也可以指定激活的profile：

	spring:
	  profiles.active: dev
      
    ---
    spring:
      profiles: dev
	myconfig:
      config1: dev-enviroment

    ---
    spring:
      profiles: test
    myconfig:
      config1: test-enviroment

    ---
    spring:
      profiles: prod
	myconfig:
      config1: prod-envioment
        

也可以在运行的执行指定profile：

	java -Dspring.profiles.active="prod" -jar yourjar.jar
    
还可以使用Profile注解，MyConfig只会在prod这个profile下才会生效，其他profile不会生效：

	@Profile("prod")
    @Component
    public class MyConfig {
		....
    }

## 自定义的一些Conveter，Interceptor ##

如果想配置springmvc的HandlerInterceptorAdapter或者HttpMessageConverter。 只需要定义自己的interceptor或者converter，然后加上Component注解。这样SpringBoot会自动处理这些类，不用自己在配置文件里指定对应的内容。 这个也是相当方便的。

	@Component
	public class AuthInterceptor extends HandlerInterceptorAdapter {
    	...
    }
    
    @Component
    public class MyConverter implements HttpMessageConverter<MyObj> { 
    	...
    }


## 模板的使用 ##

比如使用freemarker的时候，加入以下依赖：

	<dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-freemarker</artifactId>
    </dependency>
    
然后在resources目录下建立一个templates目录即可，视图将会从这个templates位置开始找。

## 其他 ##

关于其他的特性可以参考官方文档：
	
[http://docs.spring.io/spring-boot/docs/current/reference/html/](http://docs.spring.io/spring-boot/docs/current/reference/html/)

springboot还提供了一系列sample供参考：

[https://github.com/spring-projects/spring-boot/tree/master/spring-boot-samples](https://github.com/spring-projects/spring-boot/tree/master/spring-boot-samples)


