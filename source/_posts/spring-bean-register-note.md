title: Spring类注册笔记
date: 2017-06-15 22:14:37
tags:
- java
- spring
- springboot
categories: springboot

----------------

对Spring类注册功能做个笔记，包括内置的一些扫描器以及这些扫描器的用途和注意点；还有bean注册相关的接口介绍；最后就是这些扫描器在SpringBoot上的使用。

<!--more-->

## 扫描器

1. BeanDefinitionReader接口。目前有3种实现，分别是GroovyBeanDefinitionReader(groovy文件的读取器)、PropertiesBeanDefinitionReader(Properties文件的读取器)和XmlBeanDefinitionReader(xml文件的读取器)。这3个实现类都继承AbstractBeanDefinitionReader，AbstractBeanDefinitionReader抽象类内部有个BeanDefinitionRegistry接口类型的属性，BeanDefinitionRegistry接口存在的意义在于对bean数据的管理，包括bean的注册、删除、查找等。这3个实现类内部最终对bean的注册都是通过BeanDefinitionRegistry完成的，不同点在于它们处理过程不一样，比如xml文件的解析和properties文件的解析这个过程不一样
2. AnnotatedBeanDefinitionReader类。独立的一个类，用来注册单独的类，也是使用BeanDefinitionRegistry接口类型的属性完成bean的注册
3. ClassPathScanningCandidateComponentProvider类。独立的一个类，用来找出具体包下的bean信息，内部有2个TypeFilter集合属性，includeFilters和excludeFilters，分别用于对找出的bean信息做匹配，includeFilters中的TypeFilter只要有一个满足，就不会过滤；excludeFilters中的TypeFilter只要有一个满足，就会被过滤。ClassPathBeanDefinitionScanner是ClassPathScanningCandidateComponentProvider类的子类，提供了scan方法，这个scan方法会找出包下的bean信息并使用BeanDefinitionRegistry接口类型的属性完成bean的注册
4. ConfigurationClassParser类。独立的一个类，用来解析被@Configuration注解修饰的配置类。在
[SpringBoot源码分析之Spring容器的refresh过程](http://fangjian0423.github.io/2017/05/10/springboot-context-refresh/)文章中分析过ConfigurationClassParser的作用。简单点来说就是ConfigurationClassParser会解析被@Configuration注解修饰的类，然后再处理这个类内部被其它注解修饰的情况，比如@Import注解、@ComponentScan注解、@ImportResource注解、@Bean注解等。这里解析过程中也会遇到其它被@Configuration注解修饰的类，这些类会放到ConfigurationClassParser的configurationClasses属性中然后被ConfigurationClassBeanDefinitionReader处理
5. ConfigurationClassBeanDefinitionReader，独立的一个类，处理ConfigurationClassParser解析出的被@Configuration注解修饰的配置类，会处理配置类内部的被@Bean注解修饰的方法、@ImportResource注解修饰的资源、@Import注解修饰的ImportBeanDefinitionRegistrar接口。最后使用BeanDefinitionRegistry注册
6. ComponentScanAnnotationParser，独立的一个类，@ComponentScan注解对应的解析器，内部使用ClassPathBeanDefinitionScanner完成

## 扫描器注意点

1. 第4点和第5点提到的ConfigurationClassParser和ConfigurationClassBeanDefinitionReader在ConfigurationClassPostProcessor这个BeanFactoryPostProcessor中使用；它们都是跟@Configuration注解修饰的类有关系
2. AnnotatedBeanDefinitionReader构造的时候会使用AnnotationConfigUtils的registerAnnotationConfigProcessors方法预先注册一些processor bean，比如ConfigurationClassPostProcessor、AutowiredAnnotationBeanPostProcessor、RequiredAnnotationBeanPostProcessor、CommonAnnotationBeanPostProcessor
3. ClassPathBeanDefinitionScanner扫描具体包下的类，扫描完之后根据includeAnnotationConfig属性是否使用AnnotationConfigUtils的registerAnnotationConfigProcessors方法预先注册一些processor bean。includeAnnotationConfig属性默认是true，可修改
4. AnnotatedBeanDefinitionReader和ClassPathBeanDefinitionScanner加入processor的原因在于它们注册或者扫描出来的类在Spring容器的后续处理过程中进行另外的处理。比如扫描出来配置类(被@Configuration注解修饰的类)，会被ConfigurationClassPostProcessor处理(解析内部结构)，比如类中的一些被@Autowired注解修饰的属性会被AutowiredAnnotationBeanPostProcessor处理等等

## 一些接口

1. BeanDefinition接口：描述bean实例，包括属性值、构造方法的参数值、是否单例、是否抽象、作用域scope、对应的class名字等
1.1 AnnotatedBeanDefinition接口是BeanDefinition接口的子接口，包括了实例的注解信息
1.2 ScannedGenericBeanDefinition类是AnnotatedBeanDefinition接口的实现类，ClassPathScanningCandidateComponentProvider扫描出来的类信息就会封装成ScannedGenericBeanDefinition，这是一个被扫描到的类定义
1.3 AnnotatedGenericBeanDefinition类也是AnnotatedBeanDefinition接口的实现类，在AnnotatedBeanDefinitionReader读取器读取类后封装成的，这是一个被注解过的类定义
2. BeanDefinitionRegistry接口：持有BeanDefinition信息的注册中心，可以注册新的BeanDefinition、删除老的BeanDefinition、获取注册中心的BeanDefinition。这个接口是Spring提供的唯一一个可以操作BeanDefinition数据的接口
2.1 SimpleBeanDefinitionRegistry是一个简单的BeanDefinitionRegistry接口的实现类，内部使用Map<String, BeanDefinition>类型的map存储BeanDefinition信息
2.2 DefaultListableBeanFactory这个BeanFactory接口的实现类也实现了BeanDefinitionRegistry接口，内部也是使用Map<String, BeanDefinition>类型的map存储BeanDefinition信息
2.3 通用的Spring容器GenericApplicationContext也是BeanDefinitionRegistry接口的实现类，它使用内部属性DefaultListableBeanFactory完成BeanDefinitionRegistry接口的功能(DefaultListableBeanFactory实现了BeanDefinitionRegistry接口)
3. BeanFactory接口：Spring bean容器，用来管理bean的容器
4. ApplicationContext：应用程序上下文接口，我一般喜欢叫Spring容器。它不仅仅包括bean相关的操作，还包括了很多其它的东西(毕竟是跟应用程序相关的)，比如环境信息的设置、事件触发器、国际化等。 是一个全局的概念

## 扫描器在SpringBoot中的使用

SpringBoot中的SpringApplication类提供了run方法，run方法内部有一个参数叫做source，是Object类型的(有重载的方法，支持多个source，类型是Object数组)。

SpringBoot通过一个叫做BeanDefinitionLoader类的加载器去加载这些source。

```java
private int load(Object source) {
  Assert.notNull(source, "Source must not be null");
  if (source instanceof Class<?>) {
    return load((Class<?>) source);
  }
  if (source instanceof Resource) {
    return load((Resource) source);
  }
  if (source instanceof Package) {
    return load((Package) source);
  }
  if (source instanceof CharSequence) {
    return load((CharSequence) source);
  }
  throw new IllegalArgumentException("Invalid source type " + source.getClass());
}
```

目前支持4种类型的source，分别是：

1. Class类型：使用AnnotatedBeanDefinitionReader读取器加载
2. Resource类型：使用XmlBeanDefinitionReader读取器加载
3. Package类型：使用ClassPathBeanDefinitionScanner扫描器加载
4. CharSequence：识别这个字符串信息。如果是Class类型，用第1种；如果是Resource类型，用第2种；如果是Package类型，用第3种

Class类型：

```java
@SpringBootApplication
public class MyApplication {
    public static void main(String[] args) {
        // Class类型，使用AnnotatedBeanDefinitionReader加载
        // 加载完毕之后使用ConfigurationClassPostProcessor进行后续的处理
        SpringApplication.run(MyApplication.class, args);
    }
}
```

Resource类型：

```java
SpringApplication.run(
new Object[] {
        MyApplication.class
        , new ClassPathResource("beans.xml") // 使用XmlBeanDefinitionReader加载
}
, args);
```

Package类型：

```java
SpringApplication.run(
new Object[] {
        MyApplication.class
        , new ClassPathResource("beans.xml")
        , OtherBean.class.getPackage() // 使用ClassPathBeanDefinitionScanner加载
}
, args);
```
