title: Swagger在SpringBoot中的使用
date: 2016-10-09 22:11:23
tags:
- springboot
- swagger
categories: springboot

----------------

[Swagger](http://swagger.io/)是一个规范和完整的框架，用于生成、描述、调用和可视化 RESTful 风格的 Web 服务。

在SpringBoot中要使用Swagger的话，可以使用[springfox](https://github.com/springfox/springfox)。

在sbt中添加依赖即可：

	libraryDependencies += "io.springfox" % "springfox-swagger2" % "2.6.0"
	libraryDependencies += "io.springfox" % "springfox-swagger-ui" % "2.6.0"
	
查看springfox-swagger2中的源码，发现springfox并不是通过autoconfigure实现和swagger的整合的，而是基于Spring的方式Import各种bean构造Swagger。

所以本文指的Swagger在SpringBoot中的使用同样也可以在Spring中使用。
	

<!--more-->
	
要整合Spring和Swagger的话需要这么做。

1. 加上@EnableSwagger2注解
2. 构造一个Docket

以下代码就是例子：


	@EnableSwagger2
    @Configuration
    public class SwaggerConfiguration {

        @Bean
        public Docket userDocket() {
            ApiInfo apiInfo = new ApiInfo("A Simple of SpringBoot-Swagger",// 大标题
                    "two controllers: UserController and DeptController",// 小标题
                    "1.0",// 版本
                    "NO terms of service",
                    new Contact("format", "fangjian0423.github.io", "fangjian0423@gmail.com"), // 作者信息
                    "The Apache License, Version 2.0",// 开源许可证
                    "http://www.apache.org/licenses/LICENSE-2.0.html"// 许可证详情
            );
            return new Docket(DocumentationType.SWAGGER_2)
                    .select()
                    .paths(Predicates.or(PathSelectors.regex("/user/.*"), PathSelectors.regex("/dept/.*")))
                    .build()
                    .apiInfo(apiInfo);
        }

    }
    
需要注意的是path方法表示要构造的url，这里使用了or连接了 /user/.\** 和 /dept/.\** 这2个地址，这里用的是正则的匹配方式。

UserController：


	@RestController
    @RequestMapping(Array("/user"))
    class UserController {

      @ApiOperation(nickname = "test method", value = "just a test method")
      @RequestMapping(value = Array("/demo"), method = Array(RequestMethod.POST, RequestMethod.GET))
      def demo(): String = {
        "Hello Swagger"
      }

      @RequestMapping(value = Array("/get/{id}"), method = Array(RequestMethod.GET))
      def get(@PathVariable id: String): String = {
        s"get ${id}"
      }

      @RequestMapping(value = Array("/delete/{id}"), method = Array(RequestMethod.POST))
      def delete(
                  @ApiParam(name = "id", value = "the identity of user", required = true)
                  @PathVariable id: String
                  ): String = {
        s"delete ${id}"
      }

      @ApiImplicitParams(
        Array(
          new ApiImplicitParam(name = "name", value = "the name of user", required = true, paramType = "form", dataType = "string"),
          new ApiImplicitParam(name = "age", value = "the age of user", required = true, paramType = "form", dataType = "int")
        )
      )
      
      @RequestMapping(value = Array("/add"), method = Array(RequestMethod.POST))
      def add(req: HttpServletRequest): String = {
        s"${req.getParameter("name")}-${req.getParameter("age")}"
      }

    }

Swagger默认会去找被@RequestMapping注解的方法。

@ApiOperation注解用于说明接口的作用，作用在方法上，如果没有使用这个注解，会去@RequestMapping中的value和method等属性。

@ApiParam注解用来额外说明参数的meta data。

@ApiImplicitParams注解也用来额外说明参数，当参数不在方法里声明的时候，可以使用这个注解。需要注意的是这个注解只能作用在方法上。


Swagger还提供了@ApiModel、@ApiResponse、@Example等诸多注解用于说明接口的作用。


另外一个Controller: 


    @RestController
    @RequestMapping(Array("/dept"))
    class DeptController {

      @ApiOperation(nickname = "dept test method", value = "dept just a test method")
      @RequestMapping(value = Array("/demo"), method = Array(RequestMethod.POST, RequestMethod.GET))
      def demo(): String = {
        "Hello Swagger"
      }

    }
    
    
Swagger展示如下：

![image](http://7x2wh6.com1.z0.glb.clouddn.com/swagger01.png)
