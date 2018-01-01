title: SpringBoot with Scala
date: 2016-06-22 23:22:39
tags:
- springboot
- scala
categories: springboot

----------------

一般情况下，我们使用java开发springboot应用，可以使用scala开发springboot应用吗？

答案当然是可以的。

今天参考了阿福老师的这篇[Scala开发者的SpringBoot快速入门指南](http://afoo.me/posts/2015-07-21-scala-developers-springboot-guide.html)以及一位国际友人的一个[spring-boot-scala-web](https://github.com/bijukunjummen/spring-boot-scala-web) demo。

自己尝试地搭建了一下环境，发现用scala编写springboot应用这种体验也是非常赞的。

下面是具体的环境搭建流程：

<!--more-->

1.使用sbt作为构建工具，由于springboot官方只有基于maven或者gradle的构建方法，所以我们只能自己写了。

参考了springboot的maven配置，比如要开发web应用，需要1个spring-boot-starter-web模块，这个spring-boot-starter-web模块内地又引用了spring-boot-starter模块，spring-boot-starter模块是一个基础的starter模块，内部的依赖如下：

	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-autoconfigure</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-logging</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-core</artifactId>
			<exclusions>
				<exclusion>
					<groupId>commons-logging</groupId>
					<artifactId>commons-logging</artifactId>
				</exclusion>
			</exclusions>
		</dependency>
		<dependency>
			<groupId>org.yaml</groupId>
			<artifactId>snakeyaml</artifactId>
			<scope>runtime</scope>
		</dependency>
	</dependencies>

其中springb-boot模块内部使用了spring的依赖，spring-boot-autoconfigure模块内部有很多自动化配置的类，spring-boot-starter-web模块内部使用了spring的web模块，一些tomcat模块等。

我们将这3个模块加入到sbt中：

	libraryDependencies += "org.springframework.boot" % "spring-boot" % "1.3.5.RELEASE"
	libraryDependencies += "org.springframework.boot" % "spring-boot-autoconfigure" % "1.3.5.RELEASE"
	libraryDependencies += "org.springframework.boot" % "spring-boot-starter-web" % "1.3.5.RELEASE"
	
再加入jpa的依赖：

	libraryDependencies += "org.springframework.boot" % "spring-boot-starter-data-jpa" % "1.3.5.RELEASE"
	libraryDependencies += "com.h2database" % "h2" % "1.4.192"
	
2.编写domain，使用BeanProperty可以自动给field加入get，set方法。

	@Entity
	class Person(pName: String, pAge: Int) {

	  @Id
	  @GeneratedValue
	  @BeanProperty
	  var id: Long = _

	  @BeanProperty
	  var name: String = pName

	  @BeanProperty
	  var age: Int = pAge

	  def this() = this("unknown", -1)

	}

	object Person {
	  def apply(name: String, age: Int) = new Person(name, age)
	}
	
3.编写Repository，由于scala中没有接口这个概念，我们使用trait代替。
	
	@Repository
	trait PersonRepository extends CrudRepository[Person, Long]
	
4.编写Controller。利用构造器依赖注入将PersonRepository注入到属性里。

	@RestController
	class PersonController @Autowired() (
	  private val personRepository: PersonRepository
	) {

	  @RequestMapping(value = Array("/index"))
	  def index(): String = {
	    "hello springboot"
	  }

	  @RequestMapping(value = Array("/add"))
	  def add(req: HttpServletRequest): String = {
	    val p = Person(req.getParameter("name"), Some(req.getParameter("age").toInt).getOrElse(-1))
	    personRepository.save(p)
	    "ok"
	  }

	  @RequestMapping(value = Array("/getAll"))
	  def get(req: HttpServletRequest): java.util.List[Person] = {
	    personRepository.findAll().toList
	  }
	  
	  @RequestMapping(value = Array("/get/{id}"))
      def get(@PathVariable() id: Long): Person = {
        personRepository.findOne(id)
      }

	}

5.使用CommandLineRunner做一些初始化工作。

	@SpringBootApplication
	class MainClass {

	  @Bean
	  def init(personRepository: PersonRepository): CommandLineRunner = {
	    return new CommandLineRunner {
	      override def run(strings: String*): Unit = {
	        personRepository.save(Person("jim", 1))
	        personRepository.save(Person("tom", 2))
	        personRepository.save(Person("jerry", 3))
	      }
	    }
	  }

	}

6.配置文件application.properties中加入以下配置。

	spring.datasource.url=jdbc:h2:mem:AZ;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=FALSE
	spring.datasource.driverClassName=org.h2.Driver
	spring.datasource.username=sa
	spring.datasource.password=
	spring.jpa.database-platform=org.hibernate.dialect.H2Dialect

7.写一个入口对象启动应用。

	object Application extends App {
	  SpringApplication.run(classOf[MainClass]);
	}
	
测试：

	curl http://localhost:8080/index
	
	hello springboot

	curl http://localhost:8080/getPersons
	
	[{"id":1,"name":"jim","age":1},{"id":2,"name":"tom","age":2},{"id":3,"name":"jerry","age":3}]
	
	curl http://localhost:8080/get/1
	
	{"id":1,"name":"jim","age":1}
	
	curl http://localhost:8080/add\?name\=format\&age\=4
	
	ok
	
	curl http://localhost:8080/getPersons
	
	[{"id":1,"name":"jim","age":1},{"id":2,"name":"tom","age":2},{"id":3,"name":"jerry","age":3},{"id":4,"name":"format","age":4}]
	
