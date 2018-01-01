title: Spring自定义类扫描器
date: 2017-06-11 18:41:50
tags:
- java
- spring
- springboot
categories: springboot

----------------

在我们刚开始接触Spring的时候，要定义bean的话需要在xml中编写，比如：

```xml
<bean id="myBean" class="your.pkg.YourClass"/>
```

后来发现如果bean比较多，会需要写很多的bean标签，太麻烦了。于是出现了一个component-scan注解。这个注解直接指定包名就可以，它会去扫描这个包下所有的class，然后判断是否解析：

```xml
<context:component-scan base-package="your.pkg"/>
```

再后来，由于注解Annotation的流行，出现了@ComponentScan注解，作用跟component-scan标签一样，跟@Configuration注解配合使用：

```java
@ComponentScan(basePackages = {"your.pkg", "other.pkg"})
public class Application { ... }
```

不论是component-scan标签，还是@ComponentScan注解。它们扫描或解析的bean只能是Spring内部所定义的，比如@Component、@Service、@Controller或@Repository。如果有一些自定义的注解，比如@Consumer、这个注解修饰的类是不会被扫描到的。这个时候我们就得自定义扫描器完成这个操作。

<!--more-->

## Spring内置的扫描器

component-scan标签底层使用ClassPathBeanDefinitionScanner这个类完成扫描工作的。@ComponentScan注解配合@Configuration注解使用，底层使用ComponentScanAnnotationParser解析器完成解析工作。

ComponentScanAnnotationParser解析器内部使用了ClassPathBeanDefinitionScanner扫描器，ClassPathBeanDefinitionScanner扫描器内部的处理过程整理如下：

1. 遍历basePackages，根据每个basePackage找出这个包下的所有的class。比如basePackage为your/pkg，会找出your.pkg包下所有的class。找出之后封装成Resource接口集合，这个Resource接口是Spring对资源的封装，有FileSystemResource、ClassPathResource、UrlResource实现等
2. 遍历找到的Resource集合，通过includeFilters和excludeFilters判断是否解析。这里的includeFilters和excludeFilters是TypeFilter接口类型的集合，是ClassPathBeanDefinitionScanner内部的属性。TypeFilter接口是一个用于判断类型是否满足要求的类型过滤器。excludeFilters中只要有一个TypeFilter满足条件，这个Resource就会被过滤。includeFilters中只要有一个TypeFilter满足条件，这个Resource就不会被过滤
3. 如果没有被过滤。把Resource封装成ScannedGenericBeanDefinition添加到BeanDefinition结果集中
4. 返回最后的BeanDefinition结果集

TypeFilter接口的定义：

```java
public interface TypeFilter {
	boolean match(MetadataReader metadataReader, MetadataReaderFactory metadataReaderFactory)
			throws IOException;
}
```

TypeFilter接口目前有AnnotationTypeFilter实现类(类是否有注解修饰)、RegexPatternTypeFilter(类名是否满足正则表达式)等。

ClassPathBeanDefinitionScanner继承ClassPathScanningCandidateComponentProvider类。

ClassPathScanningCandidateComponentProvider内部的构造函数提供了一个useDefaultFilters参数：

```java
public ClassPathScanningCandidateComponentProvider(boolean useDefaultFilters) {
  this(useDefaultFilters, new StandardEnvironment());
}
```

useDefaultFilters这个参数表示是否使用默认的TypeFilter，如果设置为true，会添加默认的TypeFilter：

```java
protected void registerDefaultFilters() {
  this.includeFilters.add(new AnnotationTypeFilter(Component.class));
  ClassLoader cl = ClassPathScanningCandidateComponentProvider.class.getClassLoader();
  try {
    this.includeFilters.add(new AnnotationTypeFilter(
        ((Class<? extends Annotation>) ClassUtils.forName("javax.annotation.ManagedBean", cl)), false));
    logger.debug("JSR-250 'javax.annotation.ManagedBean' found and supported for component scanning");
  }
  catch (ClassNotFoundException ex) {
    // JSR-250 1.1 API (as included in Java EE 6) not available - simply skip.
  }
  try {
    this.includeFilters.add(new AnnotationTypeFilter(
        ((Class<? extends Annotation>) ClassUtils.forName("javax.inject.Named", cl)), false));
    logger.debug("JSR-330 'javax.inject.Named' annotation found and supported for component scanning");
  }
  catch (ClassNotFoundException ex) {
    // JSR-330 API not available - simply skip.
  }
}
```

我们看到这里includeFilters加上了AnnotationTypeFilter，并且对应的注解是@Component。@Service、@Controller或@Repository注解它们内部都是被@Component注解所修饰的，所以它们也会被识别。


## 自定义扫描功能

一般情况下，我们要自定义扫描功能的话，可以直接使用ClassPathScanningCandidateComponentProvider完成，加上一些自定义的TypeFilter即可。或者写个自定义扫描器继承ClassPathScanningCandidateComponentProvider，并在内部添加自定义的TypeFilter。后者相当于对前者的封装。

我们就以一个简单的例子说明一下自定义扫描的实现，直接使用ClassPathScanningCandidateComponentProvider。

项目结构如下：

		./
		└── spring
		    └── study
		        └── componentprovider
		            ├── annotation
		            │   └──  Consumer.java
		            ├── bean
		            │   ├── ConsumerWithComponentAnnotation.java
		            │   ├── ConsumerWithConsumerAnnotation.java
		            │   ├── ConsumerWithInterface.java
		            │   ├── ConsumerWithNothing.java
		            │   └── ProducerWithInterface.java
		            └── interfaze
		                ├── IConsumer.java
		                └── IProducer.java

我们直接使用ClassPathScanningCandidateComponentProvider扫描spring.study.componentprovider.bean包下的class：

```java
ClassPathScanningCandidateComponentProvider provider = new ClassPathScanningCandidateComponentProvider(false); // 不使用默认的TypeFilter
provider.addIncludeFilter(new AnnotationTypeFilter(Consumer.class));
provider.addIncludeFilter(new AssignableTypeFilter(IConsumer.class));
Set<BeanDefinition> beanDefinitionSet = provider.findCandidateComponents("spring.study.componentprovider.bean");
```

这里扫描出来的类只有2个，分别是ConsumerWithConsumerAnnotation(被@Consumer注解修饰)和ConsumerWithInterface(实现了IConsumer接口)。ConsumerWithComponentAnnotation使用@Component注解，ConsumerWithNothing没实现任何借口，没使用任何注解，ProducerWithInterface实现了IProducer接口；所以这3个类不会被识别。

如果我们要自定义ComponentProvider，继承ClassPathScanningCandidateComponentProvider类即可。

RepositoryComponentProvider这个类是SpringData模块提供的，继承自ClassPathScanningCandidateComponentProvider，主要是为了识别SpringData相关的类。

它内部定义了一些自定义TypeFilter，比如InterfaceTypeFilter(识别接口的TypeFilter，目标比较是个接口，而不是实现类)、AllTypeFilter(保存存储TypeList集合，这个集合内部所有的TypeFilter必须全部满足条件才能被识别)等。

有兴趣的读者可以查看源码。
