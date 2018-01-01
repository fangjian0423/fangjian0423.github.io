title: SpringBoot源码分析之工厂加载机制
date: 2017-06-05 19:30:30
tags:
- java
- springboot
- springboot源码分析
categories: springboot

----------------

在之前的一些文章中，我们提到过从spring.factories中找出key为XXX的类。比如@EnableAutoConfiguration注解对应的EnableAutoConfigurationImportSelector中的selectImport方法会在spring.factories文件中找出key为EnableAutoConfiguration对应的值。这些类都是自动化配置类：

    // 这个spring.factories文件在spring-boot-autoconfigure模块的 META-INF/spring.factories中
    org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
    org.springframework.boot.autoconfigure.admin.SpringApplicationAdminJmxAutoConfiguration,\
    org.springframework.boot.autoconfigure.aop.AopAutoConfiguration,\
    org.springframework.boot.autoconfigure.amqp.RabbitAutoConfiguration,\
    org.springframework.boot.autoconfigure.batch.BatchAutoConfiguration,\
    org.springframework.boot.autoconfigure.cache.CacheAutoConfiguration,\
    org.springframework.boot.autoconfigure.cassandra.CassandraAutoConfiguration,\
    org.springframework.boot.autoconfigure.cloud.CloudAutoConfiguration,\
    org.springframework.boot.autoconfigure.context.ConfigurationPropertiesAutoConfiguration,\
    org.springframework.boot.autoconfigure.context.MessageSourceAutoConfiguration,\
    org.springframework.boot.autoconfigure.context.PropertyPlaceholderAutoConfiguration,\
    org.springframework.boot.autoconfigure.couchbase.CouchbaseAutoConfiguration,\
    ......

<!--more-->

代码的实现，EnableAutoConfigurationImportSelector的selectImport方法：

```java
@Override
public String[] selectImports(AnnotationMetadata metadata) {
  try {
    // 获取注解的属性
    AnnotationAttributes attributes = getAttributes(metadata);
    // 读取spring.factories属性文件中的数据
    List<String> configurations = getCandidateConfigurations(metadata,
        attributes);
    // 删除重复的配置类
    configurations = removeDuplicates(configurations);
    // 找到@EnableAutoConfiguration注解中定义的需要被过滤的配置类
    Set<String> exclusions = getExclusions(metadata, attributes);
    // 删除这些需要被过滤的配置类
    configurations.removeAll(exclusions);
    // 配置类做排序
    configurations = sort(configurations);
    // 记录配置类的处理信息到ConditionEvaluationReport中
    recordWithConditionEvaluationReport(configurations, exclusions);
    // 返回最终得到的自动化配置类
    return configurations.toArray(new String[configurations.size()]);
  }
  catch (IOException ex) {
    throw new IllegalStateException(ex);
  }
}

protected List<String> getCandidateConfigurations(AnnotationMetadata metadata,
    AnnotationAttributes attributes) {
  // 调用SpringFactoriesLoader的loadFactoryNames静态方法
  // getSpringFactoriesLoaderFactoryClass方法返回的是EnableAutoConfiguration类对象
  return SpringFactoriesLoader.loadFactoryNames(
      getSpringFactoriesLoaderFactoryClass(), getBeanClassLoader());
}

public static List<String> loadFactoryNames(Class<?> factoryClass, ClassLoader classLoader) {
  // 解析出properties文件中需要的key值
  String factoryClassName = factoryClass.getName();
  try {
    // 常量FACTORIES_RESOURCE_LOCATION的值为META-INF/spring.factories
    // 使用类加载器找META-INF/spring.factories资源
    Enumeration<URL> urls = (classLoader != null ? classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
        ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
    List<String> result = new ArrayList<String>();
    // 遍历找到的资源
    while (urls.hasMoreElements()) {
      URL url = urls.nextElement();
      // 使用属性文件加载资源
      Properties properties = PropertiesLoaderUtils.loadProperties(new UrlResource(url));
      // 找出key为参数factoryClass类对象对应的全名称对应的值
      String factoryClassNames = properties.getProperty(factoryClassName);
      // 以逗号分隔添加到结果集中
      result.addAll(Arrays.asList(StringUtils.commaDelimitedListToStringArray(factoryClassNames)));
    }
    return result;
  }
  catch (IOException ex) {
    throw new IllegalArgumentException("Unable to load [" + factoryClass.getName() +
        "] factories from location [" + FACTORIES_RESOURCE_LOCATION + "]", ex);
  }
}
```

Spring Framework内部使用一种**工厂加载机制(Factory Loading Mechanism)**。这种机制使用SpringFactoriesLoader完成，SpringFactoriesLoader使用loadFactories方法加载并实例化从META-INF目录里的spring.factories文件出来的工厂，这些spring.factories文件都是从classpath里的jar包里找出来的。

spring.factories文件是以Java的Properties格式存在，key是接口或抽象类的全名、value是以逗号 " , " 分隔的实现类，比如：

    example.MyService=example.MyServiceImpl1,example.MyServiceImpl2

其中example.MyService是接口的全名，example.MyServiceImpl1和example.MyServiceImpl2是这个接口的两种实现。

可通过SpringFactoriesLoader完成：

```java
List<String> classes = SpringFactoriesLoader.loadFactoryNames(EnableAutoConfiguration.class, this.getClass().getClassLoader());
classes.forEach(clazz -> {
    System.out.println("==== " + clazz);
});
```

总结：

工厂加载机制是Spring内部提供的一个约定俗成的加载方式。只需要在模块的META-INF目录下定义Properties格式的spring.factories文件，这个Properties格式的文件中的key是接口或抽象类的全名，value是以逗号 " , " 分隔的实现类。

SpringBoot中的autoconfigure模块中的spring.factories就存在于META-INF目录下：

    ├── META-INF
    │   ├── MANIFEST.MF
    │   ├── additional-spring-configuration-metadata.json
    │   ├── maven
    │   │   └── org.springframework.boot
    │   │       └── spring-boot-autoconfigure
    │   │           ├── pom.properties
    │   │           └── pom.xml
    │   ├── spring-configuration-metadata.json
    │   └── spring.factories
    ├── org
    │   └── springframework
    │       └── boot
    │           └── autoconfigure
    │               ├── AbstractDependsOnBeanFactoryPostProcessor.class
    ....

而且也定义了一些配置，比如自动化配置信息：

    org.springframework.boot.autoconfigure.EnableAutoConfiguration=...

应用初始化器：

    org.springframework.context.ApplicationContextInitializer=...

应用监听器：

    org.springframework.context.ApplicationListener=...

模板可用提供器：

    org.springframework.boot.autoconfigure.template.TemplateAvailabilityProvider=...

等等。

我们只需要遵守这个机制并在对应的文件中写出需要加载的接口和实例即可，或者自己使用SpringFactoriesLoader实现加载。
