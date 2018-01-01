title: SpringBoot源码分析之内置Servlet容器
date: 2017-05-22 18:35:36
tags:
- springboot
- java
- springboot源码分析
categories: springboot

----------------

SpringBoot内置了Servlet容器，这样项目的发布、部署就不需要额外的Servlet容器，直接启动jar包即可。SpringBoot官方文档上有一个小章节[内置servlet容器支持](https://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#boot-features-embedded-container)用于说明内置Servlet的相关问题。

在[SpringBoot源码分析之SpringBoot的启动过程](http://fangjian0423.github.io/2017/04/30/springboot-startup-analysis/)文章中我们了解到如果是Web程序，那么会构造AnnotationConfigEmbeddedWebApplicationContext类型的Spring容器，在[SpringBoot源码分析之Spring容器的refresh过程](http://fangjian0423.github.io/2017/05/10/springboot-context-refresh/)文章中我们知道AnnotationConfigEmbeddedWebApplicationContext类型的Spring容器在refresh的过程中会在onRefresh方法中创建内置的Servlet容器。

接下来，我们分析一下内置的Servlet容器相关的知识点。

<!--more-->

## 内置Servlet容器相关的接口和类

SpringBoot对内置的Servlet容器做了一层封装：

    public interface EmbeddedServletContainer {
        // 启动内置的Servlet容器，如果容器已经启动，则不影响
    	void start() throws EmbeddedServletContainerException;
        // 关闭内置的Servlet容器，如果容器已经关系，则不影响
    	void stop() throws EmbeddedServletContainerException;
        // 内置的Servlet容器监听的端口
    	int getPort();
    }

它目前有3个实现类，分别是JettyEmbeddedServletContainer、TomcatEmbeddedServletContainer和UndertowEmbeddedServletContainer，分别对应Jetty、Tomcat和Undertow这3个Servlet容器。

EmbeddedServletContainerFactory接口是一个工厂接口，用于生产EmbeddedServletContainer：

    public interface EmbeddedServletContainerFactory {
        // 获得一个已经配置好的内置Servlet容器，但是这个容器还没有监听端口。需要手动调用内置Servlet容器的start方法监听端口
        // 参数是一群ServletContextInitializer，Servlet容器启动的时候会遍历这些ServletContextInitializer，并调用onStartup方法
    	EmbeddedServletContainer getEmbeddedServletContainer(
    			ServletContextInitializer... initializers);
    }

ServletContextInitializer表示Servlet初始化器，用于设置ServletContext中的一些配置，在使用EmbeddedServletContainerFactory接口的getEmbeddedServletContainer方法获取Servlet内置容器并且容器启动的时候调用onStartup方法：

    public interface ServletContextInitializer {
    	void onStartup(ServletContext servletContext) throws ServletException;
    }

EmbeddedServletContainerFactory是在EmbeddedServletContainerAutoConfiguration这个自动化配置类中被注册到Spring容器中的(前期是Spring容器中不存在EmbeddedServletContainerFactory类型的bean，可以自己定义EmbeddedServletContainerFactory类型的bean)：

    @AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE)
    @Configuration
    @ConditionalOnWebApplication // 在Web环境下才会起作用
    @Import(BeanPostProcessorsRegistrar.class) // 会Import一个内部类BeanPostProcessorsRegistrar
    public class EmbeddedServletContainerAutoConfiguration {

    	@Configuration
        // Tomcat类和Servlet类必须在classloader中存在
    	@ConditionalOnClass({ Servlet.class, Tomcat.class })
        // 当前Spring容器中不存在EmbeddedServletContainerFactory类型的实例
    	@ConditionalOnMissingBean(value = EmbeddedServletContainerFactory.class, search = SearchStrategy.CURRENT)
    	public static class EmbeddedTomcat {

    		@Bean
    		public TomcatEmbeddedServletContainerFactory tomcatEmbeddedServletContainerFactory() {
              // 上述条件注解成立的话就会构造TomcatEmbeddedServletContainerFactory这个EmbeddedServletContainerFactory
    			return new TomcatEmbeddedServletContainerFactory();
    		}

    	}

    	@Configuration
        // Server类、Servlet类、Loader类以及WebAppContext类必须在classloader中存在
    	@ConditionalOnClass({ Servlet.class, Server.class, Loader.class,
    			WebAppContext.class })
        // 当前Spring容器中不存在EmbeddedServletContainerFactory类型的实例
    	@ConditionalOnMissingBean(value = EmbeddedServletContainerFactory.class, search = SearchStrategy.CURRENT)
    	public static class EmbeddedJetty {

    		@Bean
    		public JettyEmbeddedServletContainerFactory jettyEmbeddedServletContainerFactory() {
                // 上述条件注解成立的话就会构造JettyEmbeddedServletContainerFactory这个EmbeddedServletContainerFactory
    			return new JettyEmbeddedServletContainerFactory();
    		}

    	}

    	@Configuration
        // Undertow类、Servlet类、以及SslClientAuthMode类必须在classloader中存在
    	@ConditionalOnClass({ Servlet.class, Undertow.class, SslClientAuthMode.class })
        // 当前Spring容器中不存在EmbeddedServletContainerFactory类型的实例
    	@ConditionalOnMissingBean(value = EmbeddedServletContainerFactory.class, search = SearchStrategy.CURRENT)
    	public static class EmbeddedUndertow {

    		@Bean
    		public UndertowEmbeddedServletContainerFactory undertowEmbeddedServletContainerFactory() {
                // 上述条件注解成立的话就会构造JettyEmbeddedServletContainerFactory这个EmbeddedServletContainerFactory
    			return new UndertowEmbeddedServletContainerFactory();
    		}

    	}
        // 在EmbeddedServletContainerAutoConfiguration自动化配置类中被导入，实现了BeanFactoryAware接口(BeanFactory会被自动注入进来)和ImportBeanDefinitionRegistrar接口(会被ConfigurationClassBeanDefinitionReader解析并注册到Spring容器中)
        public static class EmbeddedServletContainerCustomizerBeanPostProcessorRegistrar
      			implements ImportBeanDefinitionRegistrar, BeanFactoryAware {

      		private ConfigurableListableBeanFactory beanFactory;

      		@Override
      		public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
      			if (beanFactory instanceof ConfigurableListableBeanFactory) {
      				this.beanFactory = (ConfigurableListableBeanFactory) beanFactory;
      			}
      		}

      		@Override
      		public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata,
      				BeanDefinitionRegistry registry) {
      			if (this.beanFactory == null) {
      				return;
      			}
                  // 如果Spring容器中不存在EmbeddedServletContainerCustomizerBeanPostProcessor类型的bean
      			if (ObjectUtils.isEmpty(this.beanFactory.getBeanNamesForType(
      					EmbeddedServletContainerCustomizerBeanPostProcessor.class, true,
      					false))) {
                      // 注册一个EmbeddedServletContainerCustomizerBeanPostProcessor
      				registry.registerBeanDefinition(
      						"embeddedServletContainerCustomizerBeanPostProcessor",
      						new RootBeanDefinition(
      								EmbeddedServletContainerCustomizerBeanPostProcessor.class));

      			}
      		}

      	}

    }

EmbeddedServletContainerCustomizerBeanPostProcessor是一个BeanPostProcessor，它在postProcessBeforeInitialization过程中去寻找Spring容器中EmbeddedServletContainerCustomizer类型的bean，并依次调用EmbeddedServletContainerCustomizer接口的customize方法做一些定制化：

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName)
        throws BeansException {
      // 在Spring容器中寻找ConfigurableEmbeddedServletContainer类型的bean，SpringBoot内部的3种内置Servlet容器工厂都实现了这个接口，该接口的作用就是进行Servlet容器的配置
      // 比如添加Servlet初始化器addInitializers、添加错误页addErrorPages、设置session超时时间setSessionTimeout、设置端口setPort等等
      if (bean instanceof ConfigurableEmbeddedServletContainer) {
        postProcessBeforeInitialization((ConfigurableEmbeddedServletContainer) bean);
      }
      return bean;
    }

    private void postProcessBeforeInitialization(
        ConfigurableEmbeddedServletContainer bean) {
      for (EmbeddedServletContainerCustomizer customizer : getCustomizers()) {
        // 遍历获取的每个定制化器，并调用customize方法进行一些定制
        customizer.customize(bean);
      }
    }

    private Collection<EmbeddedServletContainerCustomizer> getCustomizers() {
      if (this.customizers == null) {
        this.customizers = new ArrayList<EmbeddedServletContainerCustomizer>(
            // 找出Spring容器中EmbeddedServletContainerCustomizer类型的bean
            this.applicationContext
                .getBeansOfType(EmbeddedServletContainerCustomizer.class,
                    false, false)
                .values());
        // 定制化器做排序
        Collections.sort(this.customizers, AnnotationAwareOrderComparator.INSTANCE);
        // 设置定制化器到属性中
        this.customizers = Collections.unmodifiableList(this.customizers);
      }
      return this.customizers;
    }

SpringBoot内置了一些EmbeddedServletContainerCustomizer，比如ErrorPageCustomizer、ServerProperties、TomcatWebSocketContainerCustomizer等。

定制器比如ServerProperties表示服务端的一些配置，以server为前缀，比如有server.port、server.contextPath、server.displayName等，它同时也实现了EmbeddedServletContainerCustomizer接口，其中customize方法的一部分代码如下：

    @Override
    public void customize(ConfigurableEmbeddedServletContainer container) {
      // 3种ServletContainerFactory都实现了ConfigurableEmbeddedServletContainer接口，所以下面的这些设置相当于对ServletContainerFactory进行设置
      // 如果配置了端口信息
      if (getPort() != null) {
        container.setPort(getPort());
      }
      ...
      // 如果配置了displayName
      if (getDisplayName() != null) {
        container.setDisplayName(getDisplayName());
      }
      // 如果配置了server.session.timeout，session超时时间。注意：这里的Session指的是ServerProperties的内部静态类Session
      if (getSession().getTimeout() != null) {
        container.setSessionTimeout(getSession().getTimeout());
      }
      ...
      // 如果使用的是Tomcat内置Servlet容器，设置对应的Tomcat配置
      if (container instanceof TomcatEmbeddedServletContainerFactory) {
        getTomcat().customizeTomcat(this,
            (TomcatEmbeddedServletContainerFactory) container);
      }
      // 如果使用的是Jetty内置Servlet容器，设置对应的Tomcat配置
      if (container instanceof JettyEmbeddedServletContainerFactory) {
        getJetty().customizeJetty(this,
            (JettyEmbeddedServletContainerFactory) container);
      }
      // 如果使用的是Undertow内置Servlet容器，设置对应的Tomcat配置
      if (container instanceof UndertowEmbeddedServletContainerFactory) {
        getUndertow().customizeUndertow(this,
            (UndertowEmbeddedServletContainerFactory) container);
      }
      // 添加SessionConfiguringInitializer这个Servlet初始化器
      // SessionConfiguringInitializer初始化器的作用是基于ServerProperties的内部静态类Session设置Servlet中session和cookie的配置
      container.addInitializers(new SessionConfiguringInitializer(this.session));
      // 添加InitParameterConfiguringServletContextInitializer初始化器
      // InitParameterConfiguringServletContextInitializer初始化器的作用是基于ServerProperties的contextParameters配置设置到ServletContext的init param中
      container.addInitializers(new InitParameterConfiguringServletContextInitializer(
          getContextParameters()));
    }

ErrorPageCustomizer在ErrorMvcAutoConfiguration自动化配置里定义，是个内部静态类：

    @Bean
    public ErrorPageCustomizer errorPageCustomizer() {
        return new ErrorPageCustomizer(this.properties);
    }

    private static class ErrorPageCustomizer
  			implements EmbeddedServletContainerCustomizer, Ordered {

    		private final ServerProperties properties;

    		protected ErrorPageCustomizer(ServerProperties properties) {
    			this.properties = properties;
    		}

    		@Override
    		public void customize(ConfigurableEmbeddedServletContainer container) {
                // 添加错误页ErrorPage，这个ErrorPage对应的路径是 /error
                // 可以通过配置修改 ${servletPath} + ${error.path}
    			container.addErrorPages(new ErrorPage(this.properties.getServletPrefix()
    					+ this.properties.getError().getPath()));
    		}

    		@Override
    		public int getOrder() {
    			return 0;
    		}

  	 }

## DispatcherServlet的构造

DispatcherServlet是SpringMVC中的核心分发器。它是在DispatcherServletAutoConfiguration这个自动化配置类里构造的(如果Spring容器内没有自定义的DispatcherServlet)，并且还会被加到Servlet容器中(通过ServletRegistrationBean完成)。

DispatcherServletAutoConfiguration这个自动化配置类存在2个条件注解@ConditionalOnWebApplication和@ConditionalOnClass(DispatcherServlet.class)都满足条件，所以会被构造(存在@AutoConfigureAfter(EmbeddedServletContainerAutoConfiguration.class)注解，会在EmbeddedServletContainerAutoConfiguration自动化配置类构造后构造)：

    @AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE)
    @Configuration
    @ConditionalOnWebApplication
    @ConditionalOnClass(DispatcherServlet.class)
    @AutoConfigureAfter(EmbeddedServletContainerAutoConfiguration.class)
    public class DispatcherServletAutoConfiguration ...

DispatcherServletAutoConfiguration有个内部类DispatcherServletConfiguration，它会构造DispatcherServlet(使用了条件类DefaultDispatcherServletCondition，如果Spring容器已经存在自定义的DispatcherServlet类型的bean，该类就不会被构造，会直接使用自定义的DispatcherServlet)：

    @Configuration
    // 条件类DefaultDispatcherServletCondition，是EmbeddedServletContainerAutoConfiguration的内部类
    // DefaultDispatcherServletCondition条件类会去Spring容器中找DispatcherServlet类型的实例，如果找到了不会构造DispatcherServletConfiguration，否则就是构造DispatcherServletConfiguration，该类内部会构造DispatcherServlet
    // 所以如果我们要自定义DispatcherServlet的话只需要自定义DispatcherServlet即可，这样DispatcherServletConfiguration内部就不会构造DispatcherServlet
    @Conditional(DefaultDispatcherServletCondition.class)
    // Servlet3.0开始才有的类，支持以编码的形式注册Servlet
    @ConditionalOnClass(ServletRegistration.class)
    // spring.mvc 为前缀的配置
    @EnableConfigurationProperties(WebMvcProperties.class)
    protected static class DispatcherServletConfiguration {

      @Autowired
      private ServerProperties server;

      @Autowired
      private WebMvcProperties webMvcProperties;

      @Autowired(required = false)
      private MultipartConfigElement multipartConfig;

      // Spring容器注册DispatcherServlet
      @Bean(name = DEFAULT_DISPATCHER_SERVLET_BEAN_NAME)
      public DispatcherServlet dispatcherServlet() {
        // 直接构造DispatcherServlet，并设置WebMvcProperties中的一些配置
        DispatcherServlet dispatcherServlet = new DispatcherServlet();
        dispatcherServlet.setDispatchOptionsRequest(
            this.webMvcProperties.isDispatchOptionsRequest());
        dispatcherServlet.setDispatchTraceRequest(
            this.webMvcProperties.isDispatchTraceRequest());
        dispatcherServlet.setThrowExceptionIfNoHandlerFound(
            this.webMvcProperties.isThrowExceptionIfNoHandlerFound());
        return dispatcherServlet;
      }

      @Bean(name = DEFAULT_DISPATCHER_SERVLET_REGISTRATION_BEAN_NAME)
      public ServletRegistrationBean dispatcherServletRegistration() {
        // 直接使用DispatcherServlet和server配置中的servletPath路径构造ServletRegistrationBean
        // ServletRegistrationBean实现了ServletContextInitializer接口，在onStartup方法中对应的Servlet注册到Servlet容器中
        // 所以这里DispatcherServlet会被注册到Servlet容器中，对应的urlMapping为server.servletPath配置
        ServletRegistrationBean registration = new ServletRegistrationBean(
            dispatcherServlet(), this.server.getServletMapping());
        registration.setName(DEFAULT_DISPATCHER_SERVLET_BEAN_NAME);
        if (this.multipartConfig != null) {
          registration.setMultipartConfig(this.multipartConfig);
        }
        return registration;
      }

      @Bean // 构造文件上传相关的bean
      @ConditionalOnBean(MultipartResolver.class)
      @ConditionalOnMissingBean(name = DispatcherServlet.MULTIPART_RESOLVER_BEAN_NAME)
      public MultipartResolver multipartResolver(MultipartResolver resolver) {
        return resolver;
      }

    }

ServletRegistrationBean实现了ServletContextInitializer接口，是个Servlet初始化器，onStartup方法代码：

    @Override
    public void onStartup(ServletContext servletContext) throws ServletException {
      Assert.notNull(this.servlet, "Servlet must not be null");
      String name = getServletName();
      if (!isEnabled()) {
        logger.info("Servlet " + name + " was not registered (disabled)");
        return;
      }
      logger.info("Mapping servlet: '" + name + "' to " + this.urlMappings);
      // 把servlet添加到Servlet容器中，Servlet容器启动的时候会加载这个Servlet
      Dynamic added = servletContext.addServlet(name, this.servlet);
      if (added == null) {
        logger.info("Servlet " + name + " was not registered "
            + "(possibly already registered?)");
        return;
      }
      // 进行Servlet的一些配置，比如urlMapping，loadOnStartup等
      configure(added);
    }

类似ServletRegistrationBean的还有ServletListenerRegistrationBean和FilterRegistrationBean，它们都是Servlet初始化器，分别都是在Servlet容器中添加Listener和Filter。


1个小漏洞：如果定义了一个名字为dispatcherServlet的bean，但是它不是DispatcherServlet类型，那么DispatcherServlet就不会被构造，@RestController和@Controller注解的控制器就没办法生效：

    @Bean(name = "dispatcherServlet")
    public Object test() {
        return new Object();
    }

## 内置Servlet容器的创建和启动

web程序对应的Spring容器是AnnotationConfigEmbeddedWebApplicationContext，继承自EmbeddedWebApplicationContext。在onRefresh方法中会去创建内置Servlet容器：

    @Override
    protected void onRefresh() {
      super.onRefresh();
      try {
        // 创建内置Servlet容器
        createEmbeddedServletContainer();
      }
      catch (Throwable ex) {
        throw new ApplicationContextException("Unable to start embedded container",
            ex);
      }
    }

    private void createEmbeddedServletContainer() {
  		EmbeddedServletContainer localContainer = this.embeddedServletContainer;
  		ServletContext localServletContext = getServletContext();
          // 内置Servlet容器和ServletContext都还没初始化的时候执行
  		if (localContainer == null && localServletContext == null) {
              // 从Spring容器中获取EmbeddedServletContainerFactory，如果EmbeddedServletContainerFactory不存在或者有多个的话会抛出异常中止程序
  			EmbeddedServletContainerFactory containerFactory = getEmbeddedServletContainerFactory();
              // 获取Servlet初始化器并创建Servlet容器，依次调用Servlet初始化器中的onStartup方法
  			this.embeddedServletContainer = containerFactory
  					.getEmbeddedServletContainer(getSelfInitializer());
  		}
          // 内置Servlet容器已经初始化但是ServletContext还没初始化的时候执行
  		else if (localServletContext != null) {
  			try {
          // 对已经存在的Servlet
          容器依次调用Servlet初始化器中的onStartup方法
  				getSelfInitializer().onStartup(localServletContext);
  			}
  			catch (ServletException ex) {
  				throw new ApplicationContextException("Cannot initialize servlet context",
  						ex);
  			}
  		}
  		initPropertySources();
  	}

getSelfInitializer方法获得的Servlet初始化器内部会去构造一个ServletContextInitializerBeans(Servlet初始化器的集合)，ServletContextInitializerBeans构造的时候会去Spring容器中查找ServletContextInitializer类型的bean，其中ServletRegistrationBean、FilterRegistrationBean、ServletListenerRegistrationBean会被找出(如果有定义)，这3种ServletContextInitializer会在onStartup方法中将Servlet、Filter、Listener添加到Servlet容器中(如果我们只定义了Servlet、Filter或者Listener，ServletContextInitializerBeans内部会调用addAdaptableBeans方法把它们包装成RegistrationBean)：

    // selfInitialize方法内部调用的getServletContextInitializerBeans方法获得ServletContextInitializerBeans
    protected Collection<ServletContextInitializer> getServletContextInitializerBeans() {
      return new ServletContextInitializerBeans(getBeanFactory());
    }

    private void addServletContextInitializerBean(String beanName,
  			ServletContextInitializer initializer, ListableBeanFactory beanFactory) {
  		if (initializer instanceof ServletRegistrationBean) {
  			Servlet source = ((ServletRegistrationBean) initializer).getServlet();
  			addServletContextInitializerBean(Servlet.class, beanName, initializer,
  					beanFactory, source);
  		}
  		else if (initializer instanceof FilterRegistrationBean) {
  			Filter source = ((FilterRegistrationBean) initializer).getFilter();
  			addServletContextInitializerBean(Filter.class, beanName, initializer,
  					beanFactory, source);
  		}
  		else if (initializer instanceof DelegatingFilterProxyRegistrationBean) {
  			String source = ((DelegatingFilterProxyRegistrationBean) initializer)
  					.getTargetBeanName();
  			addServletContextInitializerBean(Filter.class, beanName, initializer,
  					beanFactory, source);
  		}
  		else if (initializer instanceof ServletListenerRegistrationBean) {
  			EventListener source = ((ServletListenerRegistrationBean<?>) initializer)
  					.getListener();
  			addServletContextInitializerBean(EventListener.class, beanName, initializer,
  					beanFactory, source);
  		}
  		else {
  			addServletContextInitializerBean(ServletContextInitializer.class, beanName,
  					initializer, beanFactory, null);
  		}
  	}

Servlet容器创建完毕之后在finishRefresh方法中会去启动：

    @Override
    protected void finishRefresh() {
      super.finishRefresh();
      // 调用startEmbeddedServletContainer方法
      EmbeddedServletContainer localContainer = startEmbeddedServletContainer();
      if (localContainer != null) {
        // 发布EmbeddedServletContainerInitializedEvent事件
        publishEvent(
            new EmbeddedServletContainerInitializedEvent(this, localContainer));
      }
    }

    private EmbeddedServletContainer startEmbeddedServletContainer() {
          // 先得到在onRefresh方法中构造的Servlet容器embeddedServletContainer
  		EmbeddedServletContainer localContainer = this.embeddedServletContainer;
  		if (localContainer != null) {
              // 启动
  			localContainer.start();
  		}
  		return localContainer;
  	}

## 自定义Servlet、Filter、Listener

SpringBoot默认只会添加一个Servlet，也就是DispatcherServlet，如果我们想添加自定义的Servlet或者是Filter还是Listener，有以下几种方法。


1.在Spring容器中声明ServletRegistrationBean、FilterRegistrationBean或者ServletListenerRegistrationBean。原理在**DispatcherServlet的构造**章节中已经说明

    @Bean
    public ServletRegistrationBean customServlet() {
        return new ServletRegistrationBean(new CustomServlet(), "/custom");
    }

    private static class CustomServlet extends HttpServlet {
        @Override
        protected void service(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
            resp.getWriter().write("receive by custom servlet");
        }
    }

2.@ServletComponentScan注解和@WebServlet、@WebFilter以及@WebListener注解配合使用。@ServletComponentScan注解启用ImportServletComponentScanRegistrar类，是个ImportBeanDefinitionRegistrar接口的实现类，会被Spring容器所解析。ServletComponentScanRegistrar内部会解析@ServletComponentScan注解，然后会在Spring容器中注册ServletComponentRegisteringPostProcessor，是个BeanFactoryPostProcessor，会去解析扫描出来的类是不是有@WebServlet、@WebListener、@WebFilter这3种注解，有的话把这3种类型的类转换成ServletRegistrationBean、FilterRegistrationBean或者ServletListenerRegistrationBean，然后让Spring容器去解析：

    @SpringBootApplication
    @ServletComponentScan
    public class EmbeddedServletApplication { ... }

    @WebServlet(urlPatterns = "/simple")
    public class SimpleServlet extends HttpServlet {

        @Override
        protected void service(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
            resp.getWriter().write("receive by SimpleServlet");
        }

    }

3.在Spring容器中声明Servlet、Filter或者Listener。因为在ServletContextInitializerBeans内部会去调用addAdaptableBeans方法把它们包装成ServletRegistrationBean：

    @Bean(name = "dispatcherServlet")
    public DispatcherServlet myDispatcherServlet() {
        return new DispatcherServlet();
    }

## Whitelabel Error Page原理

为什么SpringBoot的程序里Controller发生了错误，我们没有进行异常的捕捉，会跳转到Whitelabel Error Page页面，这是如何实现的？

![](http://7x2wh6.com1.z0.glb.clouddn.com/embedded-servlet-container01.png)

SpringBoot內部提供了一个ErrorController叫做BasicErrorController，对应的@RequestMapping地址为 "server.error.path" 配置 或者 "error.path" 配置，这2个配置没配的话默认是/error，之前分析过ErrorPageCustomizer这个定制化器会把ErrorPage添加到Servlet容器中(这个ErrorPage的path就是上面说的那2个配置)，这样Servlet容器发生错误的时候就会访问ErrorPage配置的path，所以程序发生异常且没有被catch的话，就会走Servlet容器配置的ErrorPage。下面这段代码是BasicErrorController对应的处理请求方法：

    @RequestMapping(produces = "text/html")
    public ModelAndView errorHtml(HttpServletRequest request,
      HttpServletResponse response) {
        // 设置响应码
        response.setStatus(getStatus(request).value());
        // 设置一些信息，比如timestamp、statusCode、错误message等
        Map<String, Object> model = getErrorAttributes(request,
            isIncludeStackTrace(request, MediaType.TEXT_HTML));
        // 返回error视图
        return new ModelAndView("error", model);
    }

这里名字为error视图会被BeanNameViewResolver这个视图解析器解析，它会去Spring容器中找出name为error的View，error这个bean在ErrorMvcAutoConfiguration自动化配置类里定义，它返回了一个SpelView视图，也就是刚才见到的Whitelabel Error Page(error.whitelabel.enabled配置需要是true，否则WhitelabelErrorViewConfiguration自动化配置类不会被注册)：

    @Configuration
    @ConditionalOnProperty(prefix = "server.error.whitelabel", name = "enabled", matchIfMissing = true)
    @Conditional(ErrorTemplateMissingCondition.class)
    protected static class WhitelabelErrorViewConfiguration {

      // Whitelabel Error Page
      private final SpelView defaultErrorView = new SpelView(
          "<html><body><h1>Whitelabel Error Page</h1>"
              + "<p>This application has no explicit mapping for /error, so you are seeing this as a fallback.</p>"
              + "<div id='created'>${timestamp}</div>"
              + "<div>There was an unexpected error (type=${error}, status=${status}).</div>"
              + "<div>${message}</div></body></html>");

      @Bean(name = "error") // bean的名字是error
      @ConditionalOnMissingBean(name = "error") // 名字为error的bean不存在才会构造
      public View defaultErrorView() {
        return this.defaultErrorView;
      }

      @Bean
      @ConditionalOnMissingBean(BeanNameViewResolver.class)
      public BeanNameViewResolver beanNameViewResolver() {
        // BeanNameViewResolver会去Spring容器找对应bean的视图
        BeanNameViewResolver resolver = new BeanNameViewResolver();
        resolver.setOrder(Ordered.LOWEST_PRECEDENCE - 10);
        return resolver;
      }

    }

如果自定义了error页面，比如使用freemarker模板的话存在/templates/error.ftl页面，使用thymeleaf模板的话存在/templates/error.html页面。那么Whitelabel Error Page就不会生效了，而是会跳到这些error页面。这又是如何实现的呢?

这是因为ErrorMvcAutoConfiguration自动化配置类里的内部类  WhitelabelErrorViewConfiguration自动化配置类里有个条件类ErrorTemplateMissingCondition，它的getMatchOutcome方法：

    @Override
    public ConditionOutcome getMatchOutcome(ConditionContext context,
        AnnotatedTypeMetadata metadata) {
      // 从spring.factories文件中找出key为TemplateAvailabilityProvider为类，TemplateAvailabilityProvider用来查询视图是否可用
      List<TemplateAvailabilityProvider> availabilityProviders = SpringFactoriesLoader
          .loadFactories(TemplateAvailabilityProvider.class,
              context.getClassLoader());
      // 遍历各个TemplateAvailabilityProvider
      for (TemplateAvailabilityProvider availabilityProvider : availabilityProviders)
        // 如果error视图可用
        if (availabilityProvider.isTemplateAvailable("error",
            context.getEnvironment(), context.getClassLoader(),
            context.getResourceLoader())) {
          // 条件不生效。WhitelabelErrorViewConfiguration不会被构造
          return ConditionOutcome.noMatch("Template from "
              + availabilityProvider + " found for error view");
        }
      }
      // 条件生效。WhitelabelErrorViewConfiguration被构造
      return ConditionOutcome.match("No error template view detected");
    }

比如FreeMarkerTemplateAvailabilityProvider这个TemplateAvailabilityProvider的逻辑如下：

    public class FreeMarkerTemplateAvailabilityProvider
    		implements TemplateAvailabilityProvider {

    	@Override
    	public boolean isTemplateAvailable(String view, Environment environment,
    			ClassLoader classLoader, ResourceLoader resourceLoader) {
            // 判断是否存在freemarker包中的Configuration类，存在的话才会继续
    		if (ClassUtils.isPresent("freemarker.template.Configuration", classLoader)) {
                // 构造属性解析器
    			RelaxedPropertyResolver resolver = new RelaxedPropertyResolver(environment,
    					"spring.freemarker.");
                // 设置一些配置
    			String loaderPath = resolver.getProperty("template-loader-path",
    					FreeMarkerProperties.DEFAULT_TEMPLATE_LOADER_PATH);
    			String prefix = resolver.getProperty("prefix",
    					FreeMarkerProperties.DEFAULT_PREFIX);
    			String suffix = resolver.getProperty("suffix",
    					FreeMarkerProperties.DEFAULT_SUFFIX);
                // 查找对应的资源文件是否存在
    			return resourceLoader.getResource(loaderPath + prefix + view + suffix)
    					.exists();
    		}
    		return false;
    	}

    }

所以BeanNameViewResolver不会被构造，Whitelabel Error Page也不会构造，而是直接去找自定义的error视图。

一些测试代码： https://github.com/fangjian0423/springboot-analysis/tree/master/springboot-embedded-servlet-conatiner
