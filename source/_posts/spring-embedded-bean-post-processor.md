title: Spring内置的BeanPostProcessor总结
date: 2017-06-24 00:57:01
tags:
- spring
- java
- springboot
categories: springboot

----------------

Spring内置了一些很有用的BeanPostProcessor接口实现类。比如有AutowiredAnnotationBeanPostProcessor、RequiredAnnotationBeanPostProcessor、CommonAnnotationBeanPostProcessor、EventListenerMethodProcessor等。这些Processor会处理各自的场景。

正是有了这些processor，把bean的构造过程中的一部分功能分配给了这些processor处理，减轻了BeanFactory的负担。

而且添加一些新的功能也很方便。比如Spring Scheduling模块，只需要添加个@EnableScheduling注解，然后加个@Scheduled注解修饰的方法即可，这个Processor内部会自行处理。

<!--more-->

## ApplicationContextAwareProcessor

ApplicationContextAwareProcessor实现BeanPostProcessor接口。

Spring容器的refresh方法内部调用prepareBeanFactory方法，prepareBeanFactory方法会添加ApplicationContextAwareProcessor到BeanFactory中。这个Processor的作用在于为实现*Aware接口的bean调用该Aware接口定义的方法，并传入对应的参数。比如实现EnvironmentAware接口的bean在该Processor内部会调用EnvironmentAware接口的setEnvironment方法，并把Spring容器内部的ConfigurableEnvironment传递进去。

具体的代码：

``` java
// AbstractApplicationContext.class
beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
// ApplicationContextAwareProcessor.class
private void invokeAwareInterfaces(Object bean) {
  if (bean instanceof Aware) {
    if (bean instanceof EnvironmentAware) {
      ((EnvironmentAware) bean).setEnvironment(this.applicationContext.getEnvironment());
    }
    if (bean instanceof EmbeddedValueResolverAware) {
      ((EmbeddedValueResolverAware) bean).setEmbeddedValueResolver(
          new EmbeddedValueResolver(this.applicationContext.getBeanFactory()));
    }
    if (bean instanceof ResourceLoaderAware) {
      ((ResourceLoaderAware) bean).setResourceLoader(this.applicationContext);
    }
    if (bean instanceof ApplicationEventPublisherAware) {
      ((ApplicationEventPublisherAware) bean).setApplicationEventPublisher(this.applicationContext);
    }
    if (bean instanceof MessageSourceAware) {
      ((MessageSourceAware) bean).setMessageSource(this.applicationContext);
    }
    if (bean instanceof ApplicationContextAware) {
      ((ApplicationContextAware) bean).setApplicationContext(this.applicationContext);
    }
  }
}
```

## CommonAnnotationBeanPostProcessor

在AnnotationConfigUtils类的registerAnnotationConfigProcessors方法中被封装成RootBeanDefinition并注册到Spring容器中。registerAnnotationConfigProcessors方法在一些比如扫描类的场景下注册。比如 context:component-scan 标签或 context:annotation-config 标签的使用，或ClassPathBeanDefinitionScanner扫描器的使用、AnnotatedBeanDefinitionReader读取器的使用。

主要处理@Resource、@PostConstruct和@PreDestroy注解的实现。

在postProcessPropertyValues过程中，该processor会找出bean中被@Resource注解修饰的属性(Field)和方法(Method)，找出以后注入到bean中。

```java
// CommonAnnotationBeanPostProcessor.class
@Override
public PropertyValues postProcessPropertyValues(
    PropertyValues pvs, PropertyDescriptor[] pds, Object bean, String beanName) throws BeansException {
  // 找出bean中被@Resource注解修饰的属性(Field)和方法(Method)
  InjectionMetadata metadata = findResourceMetadata(beanName, bean.getClass(), pvs);
  try {
    // 注入到bean中
    metadata.inject(bean, beanName, pvs);
  }
  catch (Throwable ex) {
    throw new BeanCreationException(beanName, "Injection of resource dependencies failed", ex);
  }
  return pvs;
}
```

CommonAnnotationBeanPostProcessor的父类InitDestroyAnnotationBeanPostProcessor类的postProcessMergedBeanDefinition过程会找出被@PostConstruct和@PreDestroy注解修饰的方法。

```java
// InitDestroyAnnotationBeanPostProcessor.class
@Override
public void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName) {
  if (beanType != null) {
    // 找出被@PostConstruct和@PreDestroy注解修饰的方法
    LifecycleMetadata metadata = findLifecycleMetadata(beanType);
    metadata.checkConfigMembers(beanDefinition);
  }
}

@Override
public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
  LifecycleMetadata metadata = findLifecycleMetadata(bean.getClass());
  try {
    // postProcessBeforeInitialization在实例初始化之前调用
    // 这里调用了被@PostConstruct注解修饰的方法
    metadata.invokeInitMethods(bean, beanName);
  }
  catch (InvocationTargetException ex) {
    throw new BeanCreationException(beanName, "Invocation of init method failed", ex.getTargetException());
  }
  catch (Throwable ex) {
    throw new BeanCreationException(beanName, "Couldn't invoke init method", ex);
  }
  return bean;
}

@Override
public void postProcessBeforeDestruction(Object bean, String beanName) throws BeansException {
  LifecycleMetadata metadata = findLifecycleMetadata(bean.getClass());
  try {
    // postProcessBeforeDestruction在实例销毁之前调用
    // 这里调用了被@PreDestroy注解修饰的方法
    metadata.invokeDestroyMethods(bean, beanName);
  }
  catch (InvocationTargetException ex) {
    String msg = "Invocation of destroy method failed on bean with name '" + beanName + "'";
    if (logger.isDebugEnabled()) {
      logger.warn(msg, ex.getTargetException());
    }
    else {
      logger.warn(msg + ": " + ex.getTargetException());
    }
  }
  catch (Throwable ex) {
    logger.error("Couldn't invoke destroy method on bean with name '" + beanName + "'", ex);
  }
}
```

## AutowiredAnnotationBeanPostProcessor

跟CommonAnnotationBeanPostProcessor一样，在AnnotationConfigUtils类的registerAnnotationConfigProcessors方法被注册到Spring容器中。

主要处理@Autowired、@Value、@Lookup和@Inject注解的实现，处理逻辑跟CommonAnnotationBeanPostProcessor类似。

```java
// AutowiredAnnotationBeanPostProcessor.class
@Override
public PropertyValues postProcessPropertyValues(
    PropertyValues pvs, PropertyDescriptor[] pds, Object bean, String beanName) throws BeansException {
  // 找出被@Autowired、@Value以及@Inject注解修饰的属性和方法
  InjectionMetadata metadata = findAutowiringMetadata(beanName, bean.getClass(), pvs);
  try {
    // 注入到bean中
    metadata.inject(bean, beanName, pvs);
  }
  catch (Throwable ex) {
    throw new BeanCreationException(beanName, "Injection of autowired dependencies failed", ex);
  }
  return pvs;
}
```

由于@Autowired注解可以在构造器中使用，所以AutowiredAnnotationBeanPostProcessor实现了determineCandidateConstructors方法：

```java
@Override
public Constructor<?>[] determineCandidateConstructors(Class<?> beanClass, final String beanName) throws BeansException {
  ...
  for (Constructor<?> candidate : rawCandidates) { // 遍历所有的构造器
    // 找出被@Autowired注解修饰的构造器
    AnnotationAttributes ann = findAutowiredAnnotation(candidate);
    if (ann != null) {
      ...
      candidates.add(candidate);
    }
    else if (candidate.getParameterTypes().length == 0) {
      defaultConstructor = candidate;
    }
  }
  if (!candidates.isEmpty()) { // 有找到的话使用这些构造器
    ...
    candidateConstructors = candidates.toArray(new Constructor<?>[candidates.size()]);
  }
  else { // 否则使用默认的构造器
    candidateConstructors = new Constructor<?>[0];
  }
  ...
}
```

## RequiredAnnotationBeanPostProcessor

跟CommonAnnotationBeanPostProcessor一样，在AnnotationConfigUtils类的registerAnnotationConfigProcessors方法被注册到Spring容器中。

主要处理@Required注解的实现(@Required注解只能修饰方法)，在postProcessPropertyValues过程中处理：

```java
@Override
public PropertyValues postProcessPropertyValues(
    PropertyValues pvs, PropertyDescriptor[] pds, Object bean, String beanName)
    throws BeansException {

  if (!this.validatedBeanNames.contains(beanName)) { // 查看是否已经验证过
    if (!shouldSkip(this.beanFactory, beanName)) { // 查看该bean是否不会被skip，如果在BeanDefinition中有个org.springframework.beans.factory.annotation.RequiredAnnotationBeanPostProcessor.skipRequiredCheck属性，且值为true。那么这里会skip，不做required验证
      List<String> invalidProperties = new ArrayList<String>();
      for (PropertyDescriptor pd : pds) {
        if (isRequiredProperty(pd) && !pvs.contains(pd.getName())) { // 如果属性对应的set方法被@Required注解修饰并且该属性没有被设置值的话，添加到invalidProperties集合中
          invalidProperties.add(pd.getName());
        }
      }
      if (!invalidProperties.isEmpty()) { // 如果存在被@Required注解修饰的方法对应的属性，抛出BeanInitializationException异常
        throw new BeanInitializationException(buildExceptionMessage(invalidProperties, beanName));
      }
    }
    this.validatedBeanNames.add(beanName);
  }
  return pvs;
}
```

## BeanValidationPostProcessor

默认不添加，需要手动添加。主要提供对JSR-303验证的支持，内部有个boolean类型的属性afterInitialization，默认是false。如果是false，在postProcessBeforeInitialization过程中对bean进行验证，否则在postProcessAfterInitialization过程对bean进行验证。

```java
// 手动注册BeanValidationPostProcessor
@Bean
public BeanPostProcessor beanValidationPostProcessor() {
    return new BeanValidationPostProcessor();
}
```

定义一个使用JSR-303的bean：

```java
@Component
public class BeanForBeanValidation {
    @NotNull
    private String id;
    @Min(value = 10)
    private int age;
}
```

最后实例化BeanForBeanValidation的时候，BeanValidationPostProcessor起作用，在postProcessBeforeInitialization过程中发现validate不通过，抛出异常：

    Caused by: org.springframework.beans.factory.BeanInitializationException: Bean state is invalid: age - 最小不能小于10; id - 不能为null

## AbstractAutoProxyCreator

这是一个抽象类，实现了SmartInstantiationAwareBeanPostProcessor接口。主要用于aop在Spring中的应用。

```java
@Override
public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {
  // 生成缓存key
  // AbstractAutoProxyCreator内部有个Map用于存储代理类的缓存信息
  Object cacheKey = getCacheKey(beanClass, beanName);
  // targetSourcedBeans是个String集合，如果这个bean被内部的TargetSourceCreator数组属性处理过，那么targetSourcedBeans就会存储这个bean的beanName
  // 如果targetSourcedBeans内部没有包括当前beanName
  if (beanName == null || !this.targetSourcedBeans.contains(beanName)) {
    // advisedBeans属性是个Map<Object, Boolean>类型的map，key为cacheKey，value是个Boolean，如果是true，说明这个bean已经被wrap成代理类，否则还是原先的bean
    // 这里判断cacheKey是否已经被wrap成代理类，如果没有，返回null，走Spring默认的构造bean流程
    if (this.advisedBeans.containsKey(cacheKey)) {
      return null;
    }
    // isInfrastructureClass方法判断该bean是否是aop相关的bean，比如Advice、Advisor、AopInfrastructureBean
    // shouldSkip方法默认返回false，子类可覆盖。比如AspectJAwareAdvisorAutoProxyCreator子类进行了覆盖，它内部会找出Spring容器中Advisor类型的bean，然后进行遍历判断处理的bean是否是这个Advisor，如果是则过滤
    if (isInfrastructureClass(beanClass) || shouldSkip(beanClass, beanName)) {
      this.advisedBeans.put(cacheKey, Boolean.FALSE);
      return null;
    }
  }

  if (beanName != null) {
    // 遍历内部的TargetSourceCreator数组属性，根据bean信息得到TargetSource
    // 默认情况下TargetSourceCreator数组属性是空的
    TargetSource targetSource = getCustomTargetSource(beanClass, beanName);
    if (targetSource != null) {
      // 添加beanName到targetSourcedBeans中，证明这个bean被自定义的TargetSourceCreator处理过
      this.targetSourcedBeans.add(beanName);
      // 得到Advice
      Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(beanClass, beanName, targetSource);
      // 创建代理
      Object proxy = createProxy(beanClass, beanName, specificInterceptors, targetSource);
      // 添加到proxyTypes属性中
      this.proxyTypes.put(cacheKey, proxy.getClass());
      // 返回这个代理类，这样后续对该bean便不再处理，除了postProcessAfterInitialization过程
      return proxy;
    }
  }

  return null;
}
```

从这个postProcessBeforeInstantiation方法我们得出：如果使用了自定义的TargetSourceCreator，并且这个TargetSourceCreator得到了处理bean的TargetSource结果，那么直接基于这个bean和TargetSource结果构造出代理类。这个过程发生在postProcessBeforeInstantiation方法中，所以这个代理类直接代替了原本该生成的bean。如果没有使用自定义的TargetSourceCreator，那么走默认构造bean的流程。

```java
@Override
public Object getEarlyBeanReference(Object bean, String beanName) throws BeansException {
  // 生成缓存key
  Object cacheKey = getCacheKey(bean.getClass(), beanName);
  // earlyProxyReferences用来存储提前暴露的代理对象的缓存key，这里判断是否已经处理过，没处理过的话放到earlyProxyReferences里
  if (!this.earlyProxyReferences.contains(cacheKey)) {
    this.earlyProxyReferences.add(cacheKey);
  }
  return wrapIfNecessary(bean, beanName, cacheKey);
}

protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
  // 如果已经使用了自定义的TargetSourceCreator生成了代理类，直接返回这个代理类
  if (beanName != null && this.targetSourcedBeans.contains(beanName)) {
    return bean;
  }
  // 该bean已经没有被wrap成代理类，直接返回原本生成的实例
  if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
    return bean;
  }
  // 如果是处理aop自身相关的bean或者这些bean需要被skip，也直接返回这些bean
  if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
    this.advisedBeans.put(cacheKey, Boolean.FALSE);
    return bean;
  }

  // 得到Advice
  Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
  if (specificInterceptors != DO_NOT_PROXY) { // 如果被aop处理了
    // 添加到advisedBeans属性中，说明该bean已经被wrap成代理类
    this.advisedBeans.put(cacheKey, Boolean.TRUE);
    // 创建代理类
    Object proxy = createProxy(
        bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
    // 添加到proxyTypes属性中
    this.proxyTypes.put(cacheKey, proxy.getClass());
    return proxy;
  }
  // 如果没有被aop处理，添加到advisedBeans属性中，并说明不是代理类
  this.advisedBeans.put(cacheKey, Boolean.FALSE);
  return bean;
}

@Override
public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
  if (bean != null) {
    Object cacheKey = getCacheKey(bean.getClass(), beanName);
    if (!this.earlyProxyReferences.contains(cacheKey)) {
      return wrapIfNecessary(bean, beanName, cacheKey);
    }
  }
  return bean;
}
```

从上面这些方法看出，要实例化的bean会通过wrapIfNecessary进行处理，wrapIfNecessary方法会根据情况是否wrap成代理类，最终返回这个结果。getEarlyBeanReference和postProcessAfterInitialization处理流程是一样的，唯一的区别是getEarlyBeanReference是针对单例的，而postProcessAfterInitialization方法是针对prototype的，针对prototype的话，每次实例化都会wrap成代理对象，而单例的话只需要wrap一次即可。

AbstractAutoProxyCreator抽象类有基于注解的子类AnnotationAwareAspectJAutoProxyCreator。这个AnnotationAwareAspectJAutoProxyCreator会扫描出Spring容器中带有@Aspect注解的bean，然后在getAdvicesAndAdvisorsForBean方法中会根据这个aspect查看是否被拦截，如果被拦截那么就wrap成代理类。

默认情况下，AbstractAutoProxyCreator相关的BeanPostProcessor是不会注册到Spring容器中的。比如在SpringBoot中加入aop-starter之后，会触发AopAutoConfiguration自动化配置，然后将AnnotationAwareAspectJAutoProxyCreator注册到Spring容器中。

## MethodValidationPostProcessor

默认不添加，需要手动添加。支持方法级别的JSR-303规范。需要在类上加上@Validated注解，以及在方法的参数中加上验证注解，比如@Max，@Min，@NotEmpty ...。 下面这个BeanForMethodValidation就加上了@Validated注解，并且在方法validate的参数里加上的JSR-303的验证注解。

```java
@Component
@Validated
public class BeanForMethodValidation {
    public void validate(@NotEmpty String name, @Min(10) int age) {
        System.out.println("validate, name: " + name + ", age: " + age);
    }
}
```

MethodValidationPostProcessor内部使用aop完成对方法的调用。

```java
// MethodValidationPostProcessor.class
@Override
public void afterPropertiesSet() {
  // 基于validatedAnnotationType属性构造出Pointcut，这个validatedAnnotationType属性默认是@Validated注解类型，可以进行修改
  Pointcut pointcut = new AnnotationMatchingPointcut(this.validatedAnnotationType, true);
  // 基于Pointcut和Advice构造出Advisor
  this.advisor = new DefaultPointcutAdvisor(pointcut, createMethodValidationAdvice(this.validator));
}
// MethodValidationInterceptor这个Advice内部使用JSR完成方法参数的验证
protected Advice createMethodValidationAdvice(Validator validator) {
	return (validator != null ? new MethodValidationInterceptor(validator) : new MethodValidationInterceptor());
}
```

## ScheduledAnnotationBeanPostProcessor

默认不添加，使用@EnableScheduling注解后，会被注册到Spring容器中。主要使用Spring Scheduling功能对bean中使用了@Scheduled注解的方法进行调度处理。实现了BeanPostProcessor接口。

```java
@Override
public Object postProcessAfterInitialization(final Object bean, String beanName) {
  // 判断是否是代理类，如果是代理类，拿到真正的目标类
  Class<?> targetClass = AopUtils.getTargetClass(bean);
  // 判断是否已经处理过。nonAnnotatedClasses属性是个Class集合，用于存储bean对应的class是否有@Scheduled注解的方法，如果没有，则添加到这个集合中
  if (!this.nonAnnotatedClasses.contains(targetClass)) {
    // 找出class中带有@Scheduled注解的方法
    Map<Method, Set<Scheduled>> annotatedMethods = MethodIntrospector.selectMethods(targetClass,
        new MethodIntrospector.MetadataLookup<Set<Scheduled>>() {
          @Override
          public Set<Scheduled> inspect(Method method) {
            Set<Scheduled> scheduledMethods =
                AnnotationUtils.getRepeatableAnnotations(method, Scheduled.class, Schedules.class);
            return (!scheduledMethods.isEmpty() ? scheduledMethods : null);
          }
        });
    // 如果不存在@Scheduled注解的方法
    if (annotatedMethods.isEmpty()) {
      // 添加到nonAnnotatedClasses集合中。下次不用重复处理该类
      this.nonAnnotatedClasses.add(targetClass);
      if (logger.isTraceEnabled()) {
        logger.trace("No @Scheduled annotations found on bean class: " + bean.getClass());
      }
    }
    else { // 如果存在@Scheduled注解的方法
      // 遍历这些@Scheduled注解的方法
      for (Map.Entry<Method, Set<Scheduled>> entry : annotatedMethods.entrySet()) {
        Method method = entry.getKey();
        for (Scheduled scheduled : entry.getValue()) {
          // 进行调度处理
          processScheduled(scheduled, method, bean);
        }
      }
      if (logger.isDebugEnabled()) {
        logger.debug(annotatedMethods.size() + " @Scheduled methods processed on bean '" + beanName +
            "': " + annotatedMethods);
      }
    }
  }
  return bean;
}
```

## AsyncAnnotationBeanPostProcessor

默认不添加，使用@EnableAsync注解后，会被注册到Spring容器中。AsyncAnnotationBeanPostProcessor内部使用aop处理方法的调用。

```java
// AsyncAnnotationBeanPostProcessor.class
// 实现了BeanFactoryAware接口，这里会得到beanFactory
@Override
public void setBeanFactory(BeanFactory beanFactory) {
  super.setBeanFactory(beanFactory);
  // 构造一个AsyncAnnotationAdvisor
  // AsyncAnnotationAdvisor内部的Advice是AnnotationAsyncExecutionInterceptor，Pointcut会找出带有@Async的类和@Async的方法
  AsyncAnnotationAdvisor advisor = new AsyncAnnotationAdvisor(this.executor, this.exceptionHandler);
  if (this.asyncAnnotationType != null) {
    advisor.setAsyncAnnotationType(this.asyncAnnotationType);
  }
  advisor.setBeanFactory(beanFactory);
  this.advisor = advisor;
}
// AsyncExecutionInterceptor.class。 AnnotationAsyncExecutionInterceptor的父类
@Override
public Object invoke(final MethodInvocation invocation) throws Throwable {
  // 得到方法的对应类
  Class<?> targetClass = (invocation.getThis() != null ? AopUtils.getTargetClass(invocation.getThis()) : null);
  // 得到方法
  Method specificMethod = ClassUtils.getMostSpecificMethod(invocation.getMethod(), targetClass);
  final Method userDeclaredMethod = BridgeMethodResolver.findBridgedMethod(specificMethod);
  // 得到Executor线程池。如果没有在Spring容器中找到TaskExecutor类型的线程池，直接构造一个SimpleAsyncTaskExecutor
  AsyncTaskExecutor executor = determineAsyncExecutor(userDeclaredMethod);
  if (executor == null) {
    throw new IllegalStateException(
        "No executor specified and no default executor set on AsyncExecutionInterceptor either");
  }
  // 把方法在调用封装到Callable中
  Callable<Object> task = new Callable<Object>() {
    @Override
    public Object call() throws Exception {
      try {
        Object result = invocation.proceed();
        if (result instanceof Future) {
          return ((Future<?>) result).get();
        }
      }
      catch (ExecutionException ex) {
        handleError(ex.getCause(), userDeclaredMethod, invocation.getArguments());
      }
      catch (Throwable ex) {
        handleError(ex, userDeclaredMethod, invocation.getArguments());
      }
      return null;
    }
  };
  // 提交任务
  return doSubmit(task, executor, invocation.getMethod().getReturnType());
}
```

## ServletContextAwareProcessor

默认不添加，如果Spring容器是个Web容器，那么会被添加。比如GenericWebApplicationContext容器就在postProcessBeanFactory中添加了ServletContextAwareProcessor。postProcessBeanFactory方法是在Spring容器的refresh过程中被调用的。

ServletContextAwareProcessor实现了BeanPostProcessor接口，如果Spring容器中的bean实现了ServletContextAware或ServletConfigAware接口，那么会进行处理。

```java
@Override
public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
  if (getServletContext() != null && bean instanceof ServletContextAware) {
    ((ServletContextAware) bean).setServletContext(getServletContext());
  }
  if (getServletConfig() != null && bean instanceof ServletConfigAware) {
    ((ServletConfigAware) bean).setServletConfig(getServletConfig());
  }
  return bean;
}
```

## SpringBoot内部特有的BeanPostProcessor

EmbeddedServletContainerCustomizerBeanPostProcessor主要处理实现EmbeddedServletContainerCustomizer接口的bean。EmbeddedServletContainerCustomizer接口是SpringBoot提供的用于处理内置的Servlet容器的接口：

```java
public interface EmbeddedServletContainerCustomizer {
	void customize(ConfigurableEmbeddedServletContainer container);
}
```

这个EmbeddedServletContainerCustomizerBeanPostProcessor实现了BeanPostProcessor接口，处理过程：

```java
@Override
public Object postProcessBeforeInitialization(Object bean, String beanName)
    throws BeansException {
  // 处理ConfigurableEmbeddedServletContainer类型的bean
  if (bean instanceof ConfigurableEmbeddedServletContainer) {
    postProcessBeforeInitialization((ConfigurableEmbeddedServletContainer) bean);
  }
  return bean;
}
private void postProcessBeforeInitialization(
		ConfigurableEmbeddedServletContainer bean) {
  // 找出Spring容器中EmbeddedServletContainerCustomizer接口的实现类，并遍历依次调用
	for (EmbeddedServletContainerCustomizer customizer : getCustomizers()) {
		customizer.customize(bean);
	}
}
private Collection<EmbeddedServletContainerCustomizer> getCustomizers() {
	if (this.customizers == null) {
		// Look up does not include the parent context
		this.customizers = new ArrayList<EmbeddedServletContainerCustomizer>(
				this.applicationContext
						.getBeansOfType(EmbeddedServletContainerCustomizer.class,
								false, false)
						.values());
		Collections.sort(this.customizers, AnnotationAwareOrderComparator.INSTANCE);
		this.customizers = Collections.unmodifiableList(this.customizers);
	}
	return this.customizers;
}
```

SpringBoot内部EmbeddedServletContainerCustomizer接口的实现类有ServerProperties、ErrorMvcAutoConfiguration的内部类ErrorPageCustomizer等。 我们也可以实现自定义的EmbeddedServletContainerCustomizer用于修改内置Servlet容器的属性。

SpringBoot内部还有一些比如ConfigurationPropertiesBindingPostProcessor用于处于@ConfigurationProperties注解的processor、DataSourceInitializedPublisher用于发布DataSourceInitializedEvent事件等。读者可查看相关源码。
