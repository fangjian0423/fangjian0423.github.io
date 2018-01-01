title: SpringBoot源码分析之配置环境的构造过程
date: 2017-06-10 19:30:30
tags:
- java
- springboot
- springboot源码分析
categories: springboot

----------------

SpringBoot把配置文件的加载封装成了PropertySourceLoader接口，该接口的定义如下：

```java
public interface PropertySourceLoader {
	// 支持的文件后缀
	String[] getFileExtensions();
	// 把资源Resource加载成属性源PropertySource
	PropertySource<?> load(String name, Resource resource, String profile)
			throws IOException;
}
```

PropertySource是Spring对name/value键值对的封装接口。该定义了getSource()方法，这个方法会返回得到属性源的源头。比如MapPropertySource的源头就是一个Map，PropertiesPropertySource的源头就是一个Properties。

PropertySource目前的实现类有不少，比如上面提到的MapPropertySource和PropertiesPropertySource，还有RandomValuePropertySource(source是Random)、SimpleCommandLinePropertySource(source是CommandLineArgs，命令行参数)、ServletConfigPropertySource(source是ServletConfig)等等。

PropertySourceLoader接口目前有两个实现类：PropertiesPropertySourceLoader和YamlPropertySourceLoader。

PropertiesPropertySourceLoader支持从xml或properties格式的文件中加载数据。

YamlPropertySourceLoader支持从yml或者yaml格式的文件中加载数据。

<!--more-->

## Environment的构造以及PropertySource的生成

Environment接口是Spring对当前程序运行期间的环境的封装。主要提供了两大功能：profile和property(父接口PropertyResolver提供)。目前主要有StandardEnvironment、StandardServletEnvironment和MockEnvironment3种实现，分别代表普通程序、Web程序以及测试程序的环境。

下面这段代码就是SpringBoot的run方法内调用的，它会在Spring容器构造之前调用，创建环境信息：

```java
// SpringApplication.class
private ConfigurableApplicationContext createAndRefreshContext(
		SpringApplicationRunListeners listeners,
		ApplicationArguments applicationArguments) {
	ConfigurableApplicationContext context;
	// 如果是web环境，创建StandardServletEnvironment
	// 否则，创建StandardEnvironment
	// StandardServletEnvironment继承自StandardEnvironment，StandardEnvironment继承AbstractEnvironment
	// AbstractEnvironment内部有个MutablePropertySources类型的propertySources属性，用于存储多个属性源PropertySource
	// StandardEnvironment构造的时候会默认加上2个PropertySource。分别是MapPropertySource(调用System.getProperties()配置)和SystemEnvironmentPropertySource(调用System.getenv()配置)
	ConfigurableEnvironment environment = getOrCreateEnvironment();
	// 如果设置了一些启动参数args，添加基于args的SimpleCommandLinePropertySource
	// 还会配置profile信息，比如设置了spring.profiles.active启动参数，设置到环境信息中
	configureEnvironment(environment, applicationArguments.getSourceArgs());
	// 触发ApplicationEnvironmentPreparedEvent事件
	listeners.environmentPrepared(environment);
	...
}
```

在[SpringBoot源码分析之SpringBoot的启动过程](http://fangjian0423.github.io/2017/04/30/springboot-startup-analysis/)这篇文章中，我们分析过SpringApplication启动的时候会使用[工厂加载机制](http://fangjian0423.github.io/2017/06/05/springboot-factory-loading-mechanism/)初始化一些初始化器和监听器。其中org.springframework.boot.context.config.ConfigFileApplicationListener这个监听器会被加载：

		// spring-boot-version.release/META-INF/spring.factories
		org.springframework.context.ApplicationListener=\
		...
		org.springframework.boot.context.config.ConfigFileApplicationListener,\
		...

ConfigFileApplicationListener会监听SpringApplication启动的时候发生的事件，它的监听代码：

```java
@Override
public void onApplicationEvent(ApplicationEvent event) {
	// 应用环境信息准备好的时候对应的事件。此时Spring容器尚未创建，但是环境已经创建
	if (event instanceof ApplicationEnvironmentPreparedEvent) {
		onApplicationEnvironmentPreparedEvent(
				(ApplicationEnvironmentPreparedEvent) event);
	}
	// Spring容器创建完成并在refresh方法调用之前对应的事件
	if (event instanceof ApplicationPreparedEvent) {
		onApplicationPreparedEvent(event);
	}
}

private void onApplicationEnvironmentPreparedEvent(
		ApplicationEnvironmentPreparedEvent event) {
	// 使用工厂加载机制读取key为org.springframework.boot.env.EnvironmentPostProcessor的实现类
	List<EnvironmentPostProcessor> postProcessors = loadPostProcessors();
	// 加上自己。ConfigFileApplicationListener也是一个EnvironmentPostProcessor接口的实现类
	postProcessors.add(this);
	// 排序
	AnnotationAwareOrderComparator.sort(postProcessors);
	// 遍历这些EnvironmentPostProcessor，并调用postProcessEnvironment方法
	for (EnvironmentPostProcessor postProcessor : postProcessors) {
		postProcessor.postProcessEnvironment(event.getEnvironment(),
				event.getSpringApplication());
	}
}
```

ConfigFileApplicationListener也是一个EnvironmentPostProcessor接口的实现类，在这里会被调用：

```java
// ConfigFileApplicationListener的postProcessEnvironment方法
@Override
public void postProcessEnvironment(ConfigurableEnvironment environment,
		SpringApplication application) {
	// 添加属性源到环境中
	addPropertySources(environment, application.getResourceLoader());
	// 配置需要ignore的beaninfo
	configureIgnoreBeanInfo(environment);
	// 从环境中绑定一些参数到SpringApplication中
	bindToSpringApplication(environment, application);
}

protected void addPropertySources(ConfigurableEnvironment environment,
		ResourceLoader resourceLoader) {
	// 添加一个RandomValuePropertySource到环境中
	// RandomValuePropertySource是一个用于处理随机数的PropertySource，内部存储一个Random类的实例
	RandomValuePropertySource.addToEnvironment(environment);
	try {
		// 构造一个内部类Loader，并调用它的load方法
		new Loader(environment, resourceLoader).load();
	}
	catch (IOException ex) {
		throw new IllegalStateException("Unable to load configuration files", ex);
	}
}

```

内部类Loader的处理过程整理如下：

1. 创建PropertySourcesLoader。PropertySourcesLoader内部有2个属性，分别是PropertySourceLoader集合和MutablePropertySources(内部有PropertySource的集合)。最终加载完毕之后MutablePropertySources属性中的PropertySource会被添加到环境Environment中的属性源列表中。PropertySourcesLoader被构造的时候会使用工厂加载机制获得PropertySourceLoader集合(默认就2个：PropertiesPropertySourceLoader和YamlPropertySourceLoader；可以自己扩展)，然后设置到属性中
2. 获取环境信息中激活的profile(启动项目时设置的spring.profiles.active参数)。如果没设置profile，默认使用default这个profile，并添加到profiles队列中。最后会添加一个null到profiles队列中(为了获取没有指定profile的配置文件。比如环境中有application.yml和appliation-dev.yml，这个null就保证优先加载application.yml文件)
3. profiles队列取出profile数据，使用PropertySourcesLoader内部的各个PropertySourceLoader支持的后缀去目录(默认识别4种目录classpath:/[类加载目录],classpath:/config/[类加载目录下的config目录],file:./[当前目录],file:./config/[当前目录下的config目录])查找application文件名(这4个目录是默认的，可以通过启动参数spring.config.location添加新的目录，文件名可以通过启动参数spring.config.name修改)。比如目录是file:/，文件名是application，后缀为properties，那么就会查找file:/application.properties文件，如果找到，执行第4步
4. 找出的属性源文件被加载，然后添加到PropertySourcesLoader内部的PropertySourceLoader集合中。如果该属性源文件中存在spring.profiles.active配置，识别出来并加入第2步中的profiles队列，然后重复第3步
5. 第4步找到的属性源从PropertySourcesLoader中全部添加到环境信息Environment中。如果这些属性源存在defaultProperties配置，那么会添加到Environment中的属性源集合头部，否则添加到尾部

比如项目中classpath下存在application.yml文件和application-dev.yml，application.yml文件的内容如下：

		spring.profiles.active: dev

直接启动项目，开始解析，过程如下：

1. 从环境信息中找出是否设置profile，发现没有设置。 添加默认的profile - default，然后添加到队列里，最后添加null的profile。此时profiles队列中有2个元素：default和null
2. profiles队列中先拿出null的profile。然后遍历4个目录和2个PropertySourceLoader中的4个后缀(PropertiesPropertySourceLoader的properties和xml以及YamlPropertySourceLoader的yml和yaml)的application文件名。file:./config/application.properties、file:./application.properties、classpath:/config/application.properties、classpath:/application.properties、file:./config/application.xml; file:./application.xml ....
3. 找到classpath:/application.yml文件，解析成PropertySource并添加到PropertySourcesLoader里的MutablePropertySources中。由于该文件存在spring.profiles.active配置，把dev添加到profiles队列中
4. profiles队列拿出dev这个profile。由于存在profile，寻找文件的时候会带上profile，重复第3步，比如classpath:/application-dev.yml...
5. 找到classpath:/application-dev.yml文件，解析成PropertySource并添加到PropertySourcesLoader里的MutablePropertySources中
6. profiles队列拿出default这个profile。寻找文件发现没有找到。结束

这里需要注意一下一些常用的额外参数的问题，整理如下：

1. 如果启动程序的时候设置了系统参数spring.profiles.active，那么这个参数会被设置到环境信息中(由于设置了系统参数，在StandardEnvironment的钩子方法customizePropertySources中被封装成MapPropertySource并添加到Environment中)。这样PropertySourcesLoader加载的时候不会加上default这个默认profile，但是还是会读取profile为null的配置信息。spring.profiles.active支持多个profile，比如java -Dspring.profiles.active="dev,custom" -jar yourjar.jar
2. 如果设置程序参数spring.config.location，那么查找目录的时候会多出设置的目录，也支持多个目录的设置。这些会在SpringApplication里的configureEnvironment方法中被封装成SimpleCommandLinePropertySource并添加到Environment中。比如java -jar yourjar.jar --spring.config.location=classpath:/custom,file:./custom 1 2 3。有4个参数会被设置到SimpleCommandLinePropertySource中。解析文件的时候会多出2个目录，分别是classpath:/custom和file:./custom
3. 如果设置程序参数spring.config.name，那么查找的文件名就是这个参数值。原理跟spring.config.location一样，都封装到了SimpleCommandLinePropertySource中。比如java -jar yourjar.jar --spring.config.name=myfile。 这样会去查找myfile文件，而不是默认的application文件
4. 如果设置程序参数spring.profiles.active。注意这是程序参数，不是系统参数。比如java -jar yourjar.jar --spring.profiles.active=prod。会去解析prod这个profile(不论是系统参数还是程序参数，都会被封装成多个PropertySource存在于环境信息中。最终获取profile的时候会去环境信息中拿，且都可以拿到)
6. 上面说的每个profile都是在不同文件里的。不同profile也可以存在在一个文件里。因为有profile会去加载带profile的文件的同时也会去加载不带profile的文件，并解析出这个文件中spring.profiles对应的值是profile的数据。比如profile为prod，会去查找application-prod.yml文件，也会去查找application.yml文件，其中application.yml文件只会查找spring.profiles为prod的数据

比如第6点中profile.yml的数据如下：

		spring:
		    profiles: prod
		my.name: 1

		---

		spring:
		    profiles: dev
		my.name: 2

这里会解析出spring.profiles为prod的数据，也就是my.name为1的数据。


优先级的问题：由于环境信息Environment中保存的PropertySource是MutablePropertySources，那么会去配置值的时候就存在优先级的问题。比如PropertySource1和PropertySource2都存在custom.name配置，那么会从哪个PropertySource中获取这个custom.name配置呢？它会遍历内部的PropertySource列表，越在前面的PropertySource，越先获取；比如PropertySource1在PropertySource2前面，那么会先获取PropertySource1的配置。MutablePropertySources内部添加PropertySource的时候可以选择元素的位置，可以addFirst，也可以addLast，也可以自定义位置。


总结：SpringApplication启动的时候会构造环境信息Environment，如果是web环境，创建StandardServletEnvironment，否则，创建StandardEnvironment。这两种环境创建的时候都会在内部的propertySources属性中加入一些PropertySource。比如属性属性的配置信息封装成MapPropertySource，系统环境配置信息封装成SystemEnvironmentPropertySource等。这些PropertySource集合存在在环境信息中，从环境信息中读取配置的话会遍历这些PropertySource并找到相对应的配置和值。Environment构造完成之后会读取springboot相应的配置文件，从3个角度去查找：目录、文件名和profile。这3个角度有默认值，可以进行覆盖。springboot相关的配置文件读取完成之后会被封装成PropertySource并添加到环境信息中。

## @ConfigurationProperties和@EnableConfigurationProperties注解的原理

SpringBoot内部规定了一套配置和配置属性类映射规则，可以使用@ConfigurationProperties注解配合前缀属性完成属性类的读取；再通过@EnableConfigurationProperties注解设置配置类就可以把这个配置类注入进来。

比如ES的配置类ElasticsearchProperties和对应的@EnableConfigurationProperties修饰的类ElasticsearchAutoConfiguration：

```java
// 使用前缀为spring.data.elasticsearch的配置
@ConfigurationProperties(prefix = "spring.data.elasticsearch")
public class ElasticsearchProperties {
	private String clusterName = "elasticsearch";
	private String clusterNodes;
	private Map<String, String> properties = new HashMap<String, String>();
	...
}
@Configuration
@ConditionalOnClass({ Client.class, TransportClientFactoryBean.class,
		NodeClientFactoryBean.class })
// 使用@EnableConfigurationProperties注解让ElasticsearchProperties配置生效
// 这样ElasticsearchProperties就会自动注入到属性中
@EnableConfigurationProperties(ElasticsearchProperties.class)
public class ElasticsearchAutoConfiguration implements DisposableBean {
	...
	@Autowired
	private ElasticsearchProperties properties;
	...
}
```

我们分析下这个过程的实现。

@EnableConfigurationProperties注解有个属性value，是个Class数组，它会导入一个selector：EnableConfigurationPropertiesImportSelector。这个selector的selectImport方法：

```java
@Override
public String[] selectImports(AnnotationMetadata metadata) {
	// 获取@EnableConfigurationProperties注解的属性
	MultiValueMap<String, Object> attributes = metadata.getAllAnnotationAttributes(
			EnableConfigurationProperties.class.getName(), false);
	// 得到value属性，是个Class数组
	Object[] type = attributes == null ? null
			: (Object[]) attributes.getFirst("value");
	if (type == null || type.length == 0) { // 如果value属性不存在
		return new String[] {
				// 返回Registrar，Registrar内部会注册bean
				ConfigurationPropertiesBindingPostProcessorRegistrar.class
						.getName() };
	}
	// 如果value属性存在
	// 返回Registrar，Registrar内部会注册bean
	return new String[] { ConfigurationPropertiesBeanRegistrar.class.getName(),
			ConfigurationPropertiesBindingPostProcessorRegistrar.class.getName() };
}
```

ConfigurationPropertiesBeanRegistrar和ConfigurationPropertiesBindingPostProcessorRegistrar都实现了ImportBeanDefinitionRegistrar接口，会额外注册bean。

```java
// ConfigurationPropertiesBeanRegistrar的registerBeanDefinitions方法
@Override
public void registerBeanDefinitions(AnnotationMetadata metadata,
		BeanDefinitionRegistry registry) {
	// 获取@EnableConfigurationProperties注解中的属性值Class数组
	MultiValueMap<String, Object> attributes = metadata
			.getAllAnnotationAttributes(
					EnableConfigurationProperties.class.getName(), false);
	List<Class<?>> types = collectClasses(attributes.get("value"));
	// 遍历这些Class数组
	for (Class<?> type : types) {
		// 如果这个class被@ConfigurationProperties注解修饰
		// 获取@ConfigurationProperties注解中的前缀属性
		// 否则该前缀为空字符串
		String prefix = extractPrefix(type);
		// 构造bean的名字： 前缀-类全名
		// 比如ElasticsearchProperties对应的bean名字就是spring.data.elasticsearch-org.springframework.boot.autoconfigure.data.elasticsearch.ElasticsearchProperties
		String name = (StringUtils.hasText(prefix) ? prefix + "-" + type.getName()
				: type.getName());
		if (!registry.containsBeanDefinition(name)) {
			// 这个bean没被注册的话进行注册
			registerBeanDefinition(registry, type, name);
		}
	}
}

// ConfigurationPropertiesBindingPostProcessorRegistrar的registerBeanDefinitions方法
@Override
public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata,
		BeanDefinitionRegistry registry) {
	// 先判断Spring容器里是否有ConfigurationPropertiesBindingPostProcessor类型的bean
	// 由于条件里面会判断是否已经存在这个ConfigurationPropertiesBindingPostProcessor类型的bean
	// 所以实际上条件里的代码只会执行一次
	if (!registry.containsBeanDefinition(BINDER_BEAN_NAME)) {
		BeanDefinitionBuilder meta = BeanDefinitionBuilder
				.genericBeanDefinition(ConfigurationBeanFactoryMetaData.class);
		BeanDefinitionBuilder bean = BeanDefinitionBuilder.genericBeanDefinition(
				ConfigurationPropertiesBindingPostProcessor.class);
		bean.addPropertyReference("beanMetaDataStore", METADATA_BEAN_NAME);
		registry.registerBeanDefinition(BINDER_BEAN_NAME, bean.getBeanDefinition());
		registry.registerBeanDefinition(METADATA_BEAN_NAME, meta.getBeanDefinition());
	}
}
```

ConfigurationPropertiesBindingPostProcessor在ConfigurationPropertiesBindingPostProcessorRegistrar中被注册到Spring容器中，它是一个BeanPostProcessor，它的postProcessBeforeInitialization方法如下：

```java
// Spring容器中bean被实例化之前要做的事
@Override
public Object postProcessBeforeInitialization(Object bean, String beanName)
		throws BeansException {
	// 先获取bean对应的Class中的@ConfigurationProperties注解
	ConfigurationProperties annotation = AnnotationUtils
			.findAnnotation(bean.getClass(), ConfigurationProperties.class);
	// 如果@ConfigurationProperties注解，说明这是一个配置类。比如ElasticsearchProperties
	if (annotation != null) {
		// 调用postProcessBeforeInitialization方法
		postProcessBeforeInitialization(bean, beanName, annotation);
	}
	// 同样的方法使用beanName去查找
	annotation = this.beans.findFactoryAnnotation(beanName,
			ConfigurationProperties.class);
	if (annotation != null) {
		postProcessBeforeInitialization(bean, beanName, annotation);
	}
	return bean;
}

private void postProcessBeforeInitialization(Object bean, String beanName,
		ConfigurationProperties annotation) {
	Object target = bean;
	// 构造一个PropertiesConfigurationFactory
	PropertiesConfigurationFactory<Object> factory = new PropertiesConfigurationFactory<Object>(
			target);
	// 设置属性源，这里的属性源从环境信息Environment中得到
	factory.setPropertySources(this.propertySources);
	// 设置验证器
	factory.setValidator(determineValidator(bean));
	// 设置ConversionService
	factory.setConversionService(this.conversionService == null
			? getDefaultConversionService() : this.conversionService);
	if (annotation != null) {
		// 设置@ConfigurationProperties注解对应的属性到PropertiesConfigurationFactory中
		// 比如是否忽略不合法的属性ignoreInvalidFields、忽略未知的字段、忽略嵌套属性、验证器验证不合法后是否抛出异常
		factory.setIgnoreInvalidFields(annotation.ignoreInvalidFields());
		factory.setIgnoreUnknownFields(annotation.ignoreUnknownFields());
		factory.setExceptionIfInvalid(annotation.exceptionIfInvalid());
		factory.setIgnoreNestedProperties(annotation.ignoreNestedProperties());
		if (StringUtils.hasLength(annotation.prefix())) {
			// 设置前缀
			factory.setTargetName(annotation.prefix());
		}
	}
	try {
		// 绑定属性到配置类中，比如ElasticsearchProperties
		// 会使用环境信息中的属性源进行绑定
		// 这样配置类就读取到了配置文件中的配置
		factory.bindPropertiesToTarget();
	}
	catch (Exception ex) {
		String targetClass = ClassUtils.getShortName(target.getClass());
		throw new BeanCreationException(beanName, "Could not bind properties to "
				+ targetClass + " (" + getAnnotationDetails(annotation) + ")", ex);
	}
}
```

总结：SpringBoot内部规定了一套配置和配置属性类映射规则，可以使用@ConfigurationProperties注解配合前缀属性完成属性类的读取；再通过@EnableConfigurationProperties注解设置配置类就可以把这个配置类注入进来。由于这个配置类是被注入进来的，所以它肯定在Spring容器中存在；这是因为在ConfigurationPropertiesBeanRegistrar内部会注册配置类到Spring容器中，这个配置类的实例化过程在ConfigurationPropertiesBindingPostProcessor这个BeanPostProcessor完成，它会在实例化bean之前会判断bean是否被@ConfigurationProperties注解修饰，如果有，使用PropertiesConfigurationFactory从环境信息Environment中进行值的绑定。这个ConfigurationPropertiesBeanRegistrar是在使用@EnableConfigurationProperties注解的时候被创建的(通过EnableConfigurationPropertiesImportSelector)。配置类内部属性的绑定成功与否是通过环境信息Environment中的属性源PropertySource决定的。
