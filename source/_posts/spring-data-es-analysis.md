title: SpringData ES中一些底层原理的分析
date: 2017-06-02 20:33:33
tags:
- java
- elasticsearch
categories: java

----------------

之前写过一篇[SpringData ES 关于字段名和索引中的列名字不一致导致的查询问题](http://fangjian0423.github.io/2017/05/24/spring-data-es-query-problem/)，顺便深入学习下Spring Data Elasticsearch。

[Spring Data Elasticsearch](https://github.com/spring-projects/spring-data-elasticsearch)是[Spring Data](http://projects.spring.io/spring-data/)针对Elasticsearch的实现。

它跟Spring Data一样，提供了Repository接口，我们只需要定义一个新的接口并继承这个Repository接口，然后就可以注入这个新的接口使用了。

定义接口：

```java
@Repository
public interface TaskRepository extends ElasticsearchRepository<Task, String> { }
```

注入接口进行使用：

```java
@Autowired
private TaskRepository taskRepository;

....
taskRepository.save(task);
```

<!--more-->

## Repository接口的代理生成

上面的例子中TaskRepository是个接口，而我们却直接注入了这个接口并调用方法；很明显，这是错误的。

其实SpringData ES内部基于这个TaskRepository接口构造一个SimpleElasticsearchRepository，真正被注入的是这个SimpleElasticsearchRepository。

这个过程是如何实现的呢？  来分析一下。

ElasticsearchRepositoriesAutoConfiguration自动化配置类会导入ElasticsearchRepositoriesRegistrar这个ImportBeanDefinitionRegistrar。

ElasticsearchRepositoriesRegistrar继承自AbstractRepositoryConfigurationSourceSupport，是个ImportBeanDefinitionRegistrar接口的实现类，会被Spring容器调用registerBeanDefinitions进行自定义bean的注册。

ElasticsearchRepositoriesRegistrar委托给RepositoryConfigurationDelegate完成bean的解析。

整个解析过程可以分3个步骤：

1. 找出模块中的org.springframework.data.repository.Repository接口的实现类或者org.springframework.data.repository.RepositoryDefinition注解的修饰类，并会过滤掉org.springframework.data.repository.NoRepositoryBean注解的修饰类。找出后封装到RepositoryConfiguration中
2. 遍历这些RepositoryConfiguration，然后构造成BeanDefinition并注册到Spring容器中。需要注意的是这些RepositoryConfiguration会以beanClass为ElasticsearchRepositoryFactoryBean这个类的方式被注册，并把对应的Repository接口当做构造参数传递给ElasticsearchRepositoryFactoryBean，还会设置相应的属性比如elasticsearchOperations、evaluationContextProvider、namedQueries、repositoryBaseClass、lazyInit、queryLookupStrategyKey
3. ElasticsearchRepositoryFactoryBean被实例化的时候设置对应的构造参数和属性。设置完毕以后调用afterPropertiesSet方法(实现了InitializingBean接口)。在afterPropertiesSet方法内部会去创建RepositoryFactorySupport类，并进行一些初始化，比如namedQueries、repositoryBaseClass等。然后通过这个RepositoryFactorySupport的getRepository方法基于Repository接口创建出代理类，并使用AOP添加了几个MethodInterceptor

```java
// 遍历基于第1步条件得到的RepositoryConfiguration集合
for (RepositoryConfiguration<? extends RepositoryConfigurationSource> configuration : extension
    .getRepositoryConfigurations(configurationSource, resourceLoader, inMultiStoreMode)) {
  // 构造出BeanDefinitionBuilder
  BeanDefinitionBuilder definitionBuilder = builder.build(configuration);

  extension.postProcess(definitionBuilder, configurationSource);

  if (isXml) {
    // 设置elasticsearchOperations属性
    extension.postProcess(definitionBuilder, (XmlRepositoryConfigurationSource) configurationSource);
  } else {
    // 设置elasticsearchOperations属性
    extension.postProcess(definitionBuilder, (AnnotationRepositoryConfigurationSource) configurationSource);
  }

  // 使用命名策略生成bean的名字
  AbstractBeanDefinition beanDefinition = definitionBuilder.getBeanDefinition();
  String beanName = beanNameGenerator.generateBeanName(beanDefinition, registry);

  if (LOGGER.isDebugEnabled()) {
    LOGGER.debug(REPOSITORY_REGISTRATION, extension.getModuleName(), beanName,
        configuration.getRepositoryInterface(), extension.getRepositoryFactoryClassName());
  }

  beanDefinition.setAttribute(FACTORY_BEAN_OBJECT_TYPE, configuration.getRepositoryInterface());
  // 注册到Spring容器中
  registry.registerBeanDefinition(beanName, beanDefinition);
  definitions.add(new BeanComponentDefinition(beanDefinition, beanName));
}

// build方法
public BeanDefinitionBuilder build(RepositoryConfiguration<?> configuration) {

  Assert.notNull(registry, "BeanDefinitionRegistry must not be null!");
  Assert.notNull(resourceLoader, "ResourceLoader must not be null!");
  // 得到factoryBeanName，这里会使用extension.getRepositoryFactoryClassName()去获得
  // extension.getRepositoryFactoryClassName()返回的正是ElasticsearchRepositoryFactoryBean
  String factoryBeanName = configuration.getRepositoryFactoryBeanName();
  factoryBeanName = StringUtils.hasText(factoryBeanName) ? factoryBeanName
      : extension.getRepositoryFactoryClassName();
  // 基于factoryBeanName构造BeanDefinitionBuilder
  BeanDefinitionBuilder builder = BeanDefinitionBuilder.rootBeanDefinition(factoryBeanName);

  builder.getRawBeanDefinition().setSource(configuration.getSource());
  // 设置ElasticsearchRepositoryFactoryBean的构造参数，这里是对应的Repository接口
  // 设置一些的属性值
  builder.addConstructorArgValue(configuration.getRepositoryInterface());
  builder.addPropertyValue("queryLookupStrategyKey", configuration.getQueryLookupStrategyKey());
  builder.addPropertyValue("lazyInit", configuration.isLazyInit());
  builder.addPropertyValue("repositoryBaseClass", configuration.getRepositoryBaseClassName());

  NamedQueriesBeanDefinitionBuilder definitionBuilder = new NamedQueriesBeanDefinitionBuilder(
      extension.getDefaultNamedQueryLocation());

  if (StringUtils.hasText(configuration.getNamedQueriesLocation())) {
    definitionBuilder.setLocations(configuration.getNamedQueriesLocation());
  }

  builder.addPropertyValue("namedQueries", definitionBuilder.build(configuration.getSource()));
  // 查找是否有对应Repository接口的自定义实现类
  String customImplementationBeanName = registerCustomImplementation(configuration);
  // 存在自定义实现类的话，设置到属性中
  if (customImplementationBeanName != null) {
    builder.addPropertyReference("customImplementation", customImplementationBeanName);
    builder.addDependsOn(customImplementationBeanName);
  }

  RootBeanDefinition evaluationContextProviderDefinition = new RootBeanDefinition(
      ExtensionAwareEvaluationContextProvider.class);
  evaluationContextProviderDefinition.setSource(configuration.getSource());
  // 设置一些的属性值
  builder.addPropertyValue("evaluationContextProvider", evaluationContextProviderDefinition);

  return builder;
}

// RepositoryFactorySupport的getRepository方法，获得Repository接口的代理类
public <T> T getRepository(Class<T> repositoryInterface, Object customImplementation) {

  // 获取Repository的元数据
  RepositoryMetadata metadata = getRepositoryMetadata(repositoryInterface);
  // 获取Repository的自定义实现类
  Class<?> customImplementationClass = null == customImplementation ? null : customImplementation.getClass();
  // 根据元数据和自定义实现类得到Repository的RepositoryInformation信息类
  // 获取信息类的时候如果发现repositoryBaseClass是空的话会根据meta中的信息去自动匹配
  // 具体匹配过程在下面的getRepositoryBaseClass方法中说明
  RepositoryInformation information = getRepositoryInformation(metadata, customImplementationClass);
  // 验证
  validate(information, customImplementation);
  // 得到最终的目标类实例，会通过repositoryBaseClass去查找
  Object target = getTargetRepository(information);

  // 创建代理工厂
  ProxyFactory result = new ProxyFactory();
  result.setTarget(target);
  result.setInterfaces(new Class[] { repositoryInterface, Repository.class });
  // 进行aop相关的设置
  result.addAdvice(SurroundingTransactionDetectorMethodInterceptor.INSTANCE);
  result.addAdvisor(ExposeInvocationInterceptor.ADVISOR);

  if (TRANSACTION_PROXY_TYPE != null) {
    result.addInterface(TRANSACTION_PROXY_TYPE);
  }
  // 使用RepositoryProxyPostProcessor处理
  for (RepositoryProxyPostProcessor processor : postProcessors) {
    processor.postProcess(result, information);
  }

  if (IS_JAVA_8) {
    // 如果是JDK8的话，添加DefaultMethodInvokingMethodInterceptor
    result.addAdvice(new DefaultMethodInvokingMethodInterceptor());
  }

  // 添加QueryExecutorMethodInterceptor
  result.addAdvice(new QueryExecutorMethodInterceptor(information, customImplementation, target));
  // 使用代理工厂创建出代理类，这里是使用jdk内置的代理模式
  return (T) result.getProxy(classLoader);
}

// 目标类的获取
protected Class<?> getRepositoryBaseClass(RepositoryMetadata metadata) {
  // 如果Repository接口属于QueryDsl，抛出异常。目前还不支持
  if (isQueryDslRepository(metadata.getRepositoryInterface())) {
    throw new IllegalArgumentException("QueryDsl Support has not been implemented yet.");
  }
  // 如果主键是数值类型的话，repositoryBaseClass为NumberKeyedRepository
  if (Integer.class.isAssignableFrom(metadata.getIdType())
      || Long.class.isAssignableFrom(metadata.getIdType())
      || Double.class.isAssignableFrom(metadata.getIdType())) {
    return NumberKeyedRepository.class;
  } else if (metadata.getIdType() == String.class) {
    // 如果主键是String类型的话，repositoryBaseClass为SimpleElasticsearchRepository
    return SimpleElasticsearchRepository.class;
  } else if (metadata.getIdType() == UUID.class) {
    // 如果主键是UUID类型的话，repositoryBaseClass为UUIDElasticsearchRepository
    return UUIDElasticsearchRepository.class;
  } else {
    // 否则报错
    throw new IllegalArgumentException("Unsupported ID type " + metadata.getIdType());
  }
}
```

ElasticsearchRepositoryFactoryBean是一个FactoryBean接口的实现类，getObject方法返回的上面提到的getRepository方法返回的代理对象；getObjectType方法返回的是对应Repository接口类型。

我们文章一开始提到的注入TaskRepository的时候，实际上这个对象是ElasticsearchRepositoryFactoryBean类型的实例，只不过ElasticsearchRepositoryFactoryBean实现了FactoryBean接口，所以注入的时候会得到一个代理对象，这个代理对象是由jdk内置的代理生成的，并且它的target对象是SimpleElasticsearchRepository(主键是String类型)。

## SpringData ES中ElasticsearchOperations的介绍

ElasticsearchTemplate实现了ElasticsearchOperations接口。

ElasticsearchOperations接口是SpringData对Elasticsearch操作的一层封装，比如有创建索引createIndex方法、获取索引的设置信息getSetting方法、查询对象queryForObject方法、分页查询方法queryForPage、删除文档delete方法、更新文档update方法等等。

ElasticsearchTemplate是具体的实现类，它有这些属性：

```java
// elasticsearch提供的基于java的客户端连接接口。java对es集群的操作使用这个接口完成
private Client client;
// 一个转换器接口，定义了2个方法，分别可以获得MappingContext和ConversionService
// MappingContext接口用于获取所有的持久化实体和这些实体的属性
// ConversionService目前在SpringData ES中没有被使用
private ElasticsearchConverter elasticsearchConverter;
// 内部使用EntityMapper完成对象到json字符串和json字符串到对象的映射。默认使用jackson完成映射，可自定义
private ResultsMapper resultsMapper;
// 查询超时时间
private String searchTimeout;
```

Client接口在ElasticsearchAutoConfiguration自动化配置类里被构造：

```java
@Bean
@ConditionalOnMissingBean
public Client elasticsearchClient() {
  try {
    return createClient();
  }
  catch (Exception ex) {
    throw new IllegalStateException(ex);
  }
}
```

ElasticsearchTemplate、ElasticsearchConverter以及SimpleElasticsearchMappingContext在ElasticsearchDataAutoConfiguration自动化配置类里被构造：

```java
@Bean
@ConditionalOnMissingBean
public ElasticsearchTemplate elasticsearchTemplate(Client client,
    ElasticsearchConverter converter) {
  try {
    return new ElasticsearchTemplate(client, converter);
  }
  catch (Exception ex) {
    throw new IllegalStateException(ex);
  }
}

@Bean
@ConditionalOnMissingBean
public ElasticsearchConverter elasticsearchConverter(
    SimpleElasticsearchMappingContext mappingContext) {
  return new MappingElasticsearchConverter(mappingContext);
}

@Bean
@ConditionalOnMissingBean
public SimpleElasticsearchMappingContext mappingContext() {
  return new SimpleElasticsearchMappingContext();
}
```

需要注意的是这个bean被自动化配置类构造的前提是它们在Spring容器中并不存在。

## Repository的调用过程

以自定义的TaskRepository的save方法为例，大致的执行流程如下所示：

![](http://7x2wh6.com1.z0.glb.clouddn.com/SpringData-ES-seq.png)

SimpleElasticsearchRepository的save方法具体的分析在[SpringData ES 关于字段名和索引中的列名字不一致导致的查询问题](http://fangjian0423.github.io/2017/05/24/spring-data-es-query-problem/)中分析过。

像自定义的Repository查询方法，或者Repository接口的自定义实现类的操作这些底层，可以去QueryExecutorMethodInterceptor中查看，本文就不做具体分析了。
