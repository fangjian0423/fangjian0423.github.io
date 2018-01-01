title: SpringCloud网关服务zuul介绍
date: 2017-02-22 21:33:22
tags:
- springcloud
- springboot
categories: springcloud

----------------

[Zuul](https://github.com/Netflix/zuul)是Netflix开发的一款提供动态路由、监控、弹性、安全的网关服务。

使用Zuul网关服务带来的好处是统一向外系统提供REST API，并额外提供了权限控制、负载均衡等功能，并且这些功能是从原先的服务中抽离出来并单独存在的。

Zuul提供了不同类型的filter用于处理请求，这些filter可以让我们实现以下功能：

1. 权限控制和安全性：可以识别认证需要的信息和拒绝不满足条件的请求
2. 监控：监控请求信息
3. 动态路由：根据需要动态地路由请求到后台的不同集群
4. 压力测试
5. 负载均衡
6. 静态资源处理：直接在zuul处理静态资源的响应而不需要转发这些请求到内部集群中

<!--more-->

## Zuul的执行过程介绍

Zuul基于Servlet实现，它封装了Servlet提供的相关接口，并提供了一个全新的api。

**ZuulFilter**是一个基础的抽象类，定义了一些抽象方法：

1. filterType方法: filter的类型，有"pre", "route", "post", "error", "static"
2. filterOrder方法：优先级，级别越高，越快被执行
3. shouldFilter方法：开关，如果是true，run方法会执行，否则不会执行
4. run方法：filter执行的逻辑操作

**ZuulServlet**是一个继承自HttpServlet的子类，使用Zuul所有的请求都会被这个Servlet接收并处理。

ZuulServlet覆盖了HttpServlet的service方法，所以不论是get/post/put/delete等方法都会执行相同的操作：

    @Override
    public void service(javax.servlet.ServletRequest servletRequest, javax.servlet.ServletResponse servletResponse) throws ServletException, IOException {
        try {
            init((HttpServletRequest) servletRequest, (HttpServletResponse) servletResponse); // 初始化ZuulRunner，也就是包装request和response，并设置到RequestContext中，RequestContext使用ThreadLocal获得，每个线程独立保存一份，用于存储各种信息，比如request，response，监控信息，异常信息，成功信息，执行时间等等

            // Marks this request as having passed through the "Zuul engine", as opposed to servlets
            // explicitly bound in web.xml, for which requests will not have the same data attached
            RequestContext context = RequestContext.getCurrentContext();
            context.setZuulEngineRan();

            try {
                preRoute(); // 执行 pre 类型的filter
            } catch (ZuulException e) {
                error(e); // pre 类型的filter执行报错的话执行 error 类型的filter
                postRoute(); // 执行 post 类型的filter
                return;
            }
            try {
                route(); // pre 类型的filter执行成功后，执行 route 类型的filter
            } catch (ZuulException e) {
                error(e); //route 类型的filter执行报错的话执行 error 类型的filter
                postRoute(); // 执行 post 类型的filter
                return;
            }
            try {
                postRoute(); // route 类型的filter执行成功后，执行 post 类型的filter
            } catch (ZuulException e) {
                error(e); //post 类型的filter执行报错的话执行 error 类型的filter
                return;
            }

        } catch (Throwable e) {
            error(new ZuulException(e, 500, "UNHANDLED_EXCEPTION_" + e.getClass().getName())); // 发生其他没有catch的错误的话，执行 error 类型的filter
        } finally {
            RequestContext.getCurrentContext().unset();
        }
    }

下图是[zuul wiki](https://github.com/Netflix/zuul/wiki/How-it-Works)上对filter的执行过程说明。

![filter](https://camo.githubusercontent.com/4eb7754152028cdebd5c09d1c6f5acc7683f0094/687474703a2f2f6e6574666c69782e6769746875622e696f2f7a75756c2f696d616765732f7a75756c2d726571756573742d6c6966656379636c652e706e67)

从上面的service方法中我们也可以得出：先执行pre类型的filter；如果pre filter执行失败那么执行error和post类型的filter，pre filter执行成功的话执行route类型的filter；如果route filter执行失败那么执行error和post类型的filter，route filter执行成功的话执行post filter；如果post filter执行失败那么执行error类型的filter，post filter执行成功的话，结束。上述过程中执行失败指的是ZuulException被catch，如果是其他Exception的话，那么执行error类型的filter，然后结束。


ZuulServlet里的preRoute(), route(), postRoute(), error()方法详情：

    ZuulRunner.java
    public void preRoute() throws ZuulException {
        FilterProcessor.getInstance().preRoute();
    }

    FilterProcessor.java
    public void preRoute() throws ZuulException {
        try {
            runFilters("pre");
        } catch (Throwable e) {
            if (e instanceof ZuulException) {
                throw (ZuulException) e;
            }
            throw new ZuulException(e, 500, "UNCAUGHT_EXCEPTION_IN_PRE_FILTER_" + e.getClass().getName());
        }
    }

    public Object runFilters(String sType) throws Throwable {
        if (RequestContext.getCurrentContext().debugRouting()) {
            Debug.addRoutingDebug("Invoking {" + sType + "} type filters");
        }
        boolean bResult = false;
        List<ZuulFilter> list = FilterLoader.getInstance().getFiltersByType(sType); // 获得对应类型的filter集合
        if (list != null) {
            for (int i = 0; i < list.size(); i++) { // 遍历这些类型的filter集合
                ZuulFilter zuulFilter = list.get(i);
                Object result = processZuulFilter(zuulFilter); // 调用processZuulFilter方法
                if (result != null && result instanceof Boolean) {
                    bResult |= ((Boolean) result);
                }
            }
        }
        return bResult;
    }

    public Object processZuulFilter(ZuulFilter filter) throws ZuulException {

        RequestContext ctx = RequestContext.getCurrentContext();
        boolean bDebug = ctx.debugRouting();
        final String metricPrefix = "zuul.filter-";
        long execTime = 0; // 执行时间
        String filterName = "";
        try {
            long ltime = System.currentTimeMillis(); // 执行前的时间
            filterName = filter.getClass().getSimpleName(); // 获取filter名字

            RequestContext copy = null;
            Object o = null;
            Throwable t = null;

            if (bDebug) {
                Debug.addRoutingDebug("Filter " + filter.filterType() + " " + filter.filterOrder() + " " + filterName);
                copy = ctx.copy();
            }

            ZuulFilterResult result = filter.runFilter(); // 调用ZuulFilter的runFilter方法得到ZuulFilterResult，这个类是对filter执行结果的包装，包括返回值、异常信息、状态
            ExecutionStatus s = result.getStatus(); // 得到ZuulFilterResult的状态信息
            execTime = System.currentTimeMillis() - ltime; // 得到filter的执行时间

            switch (s) { // 针对不同的ZuulFilterResult的状态做不同处理
                case FAILED: // 如果是FAILED状态，说错run方法执行失败了
                    t = result.getException(); // 得到失败的异常
                    ctx.addFilterExecutionSummary(filterName, ExecutionStatus.FAILED.name(), execTime); // 失败信息加到RequestContext中
                    break;
                case SUCCESS: // 如果是SUCCESS状态，说明run方法正确执行完毕
                    o = result.getResult(); // 得到run方法返回的结果
                    ctx.addFilterExecutionSummary(filterName, ExecutionStatus.SUCCESS.name(), execTime); // 成功信息加到RequestContext中
                    if (bDebug) {
                        Debug.addRoutingDebug("Filter {" + filterName + " TYPE:" + filter.filterType() + " ORDER:" + filter.filterOrder() + "} Execution time = " + execTime + "ms");
                        Debug.compareContextState(filterName, copy);
                    }
                    break;
                default: // 其他状态的话不做处理
                    break;
            }

            if (t != null) throw t; // 如果是FAILED状态，抛出这个Exception

            usageNotifier.notify(filter, s); // 记录监控信息
            return o;

        } catch (Throwable e) { // 如果发生了一些其它没有catch的异常
            if (bDebug) {
                Debug.addRoutingDebug("Running Filter failed " + filterName + " type:" + filter.filterType() + " order:" + filter.filterOrder() + " " + e.getMessage());
            }
            usageNotifier.notify(filter, ExecutionStatus.FAILED); // 记录监控信息
            if (e instanceof ZuulException) {
                throw (ZuulException) e;
            } else { // 封装成ZuulException并抛出
                ZuulException ex = new ZuulException(e, "Filter threw Exception", 500, filter.filterType() + ":" + filterName);
                ctx.addFilterExecutionSummary(filterName, ExecutionStatus.FAILED.name(), execTime);
                throw ex;
            }
        }
    }

    ZuulFilter.java
    public ZuulFilterResult runFilter() {
        ZuulFilterResult zr = new ZuulFilterResult();
        if (!isFilterDisabled()) { // 如果对应的zuul filter没有被disable
            if (shouldFilter()) { // shouldFilter开关是否开启
                Tracer t = TracerFactory.instance().startMicroTracer("ZUUL::" + this.getClass().getSimpleName()); // 设置监控信息
                try {
                    Object res = run(); // 调用ZuulFilter的run方法
                    zr = new ZuulFilterResult(res, ExecutionStatus.SUCCESS); // 把结果封装到ZuulFilterResult中，并设置状态为SUCCESS
                } catch (Throwable e) { // 如果发生了异常
                    t.setName("ZUUL::" + this.getClass().getSimpleName() + " failed"); // 完善监控信息
                    zr = new ZuulFilterResult(ExecutionStatus.FAILED); // 把ZuulFilterResult状态设置为FAILED
                    zr.setException(e); // 设置异常信息
                } finally {
                    t.stopAndLog();
                }
            } else {
                zr = new ZuulFilterResult(ExecutionStatus.SKIPPED); // zuul filter被disable的话，把ZuulFilterResult状态设置为SKIPPED
            }
        }
        return zr;
    }


## 在SpringCloud中使用Zuul

在SpringCloud中使用Zuul，加上@EnableZuulProxy注解，这个注解会import ZuulProxyConfiguration配置类。ZuulProxyConfiguration配置类继承ZuulConfiguration类，ZuulConfiguration配置类使用zuul开头的配置。

在这个例子中，本地端口2222有了compute-service服务。这个zuul的服务地址暴露在7777端口下。

我们定义了一个规则：

    zuul.routes.api-a-url.path=/api-a-url/**
    zuul.routes.api-a-url.url=http://localhost:2222/

这个routes对应的类型是Map<String, ZuulRoute>，key为String，value是一个ZuulRoute。ZuulRoute中定义了一些属性，有：

    private String id; // 标识一个路由规则
    private String path; // 拦截路径，比如 /api-a-url/**
    private String serviceId; // Eureka服务发现中的serviceId
    private String url; //不使用服务发现中的服务，独立的一个url
    private boolean stripPrefix = true;
    private Boolean retryable; // 是否会retry
    private Set<String> sensitiveHeaders = new LinkedHashSet<>();

上面的api-a-url就是对应map中的key，path和url对应ZuulRoute中的path和url属性。


在ZuulProxyConfiguration配置类中，构造了很多bean，比如有ZuulController、ZuulHandlerMapping、DiscoveryClientRouteLocator、各种filter等bean。

其中ZuulController内部使用了ZuulServlet处理http请求，DiscoveryClientRouteLocator使用ZuulProperties中的route解析路由规则，然后封装成org.springframework.cloud.netflix.zuul.filters.Route在getRoutes方法中返回，这个方法会在RoutesEndpoint和ZuulHandlerMapping中被调用。

另外DiscoveryClientRouteLocator会基于服务发现中心中的服务信息，再去寻找对应的路由规则。由于例子中有个本地端口为2222的compute-service服务。所以会被解析并放到路由规则里，这样路由规则里就有2个规则：

1. path为/api-a-url/**，url为http://localhost:2222/
2. path为/compute-service/**，serviceId为compute-service


ZuulHandlerMapping是一个HandlerMapping，用于处理请求的映射关系。在SpringMVC中，默认是使用RequestMappingHandlerMapping处理，而在Zuul中，使用ZuulHandlerMapping处理地址映射关系。它内部有个注册handler方法：

    private void registerHandlers() {
      Collection<Route> routes = this.routeLocator.getRoutes(); // 得到路由规则
      if (routes.isEmpty()) {
        this.logger.warn("No routes found from RouteLocator");
      }
      else {
        for (Route route : routes) { // 注册路由规则中的地址，对应了handler是zuul属性，这个zuul也就是ZuulController
          registerHandler(route.getFullPath(), this.zuul);
        }
      }
    }

例子中路由规则里对应的路径有2个，分别是/api-a-url/**和/compute-service/**，它们对应的handler都是ZuulController。


访问地址：

    http://localhost:7777/api-a-url/add?a=1&b=2

在ZuulHandlerMapping中的规则路径中发现了/api-a-url/**，于是传递给ZuulController处理，ZuulController传递给ZuulServlet处理。


讲到这里，细心的读者可能会发现一个问题：**我们前面讲了这么多关于filter的各种细节，但是真正的服务调用是在哪里执行的?**

Zuul把真正的服务调用也放在了filter中处理，并在产生的结果放在了RequestContext中。

其中有route类型的filter中使用HttpClient执行，执行结果的stream放到了RequestContext。

post类型的filter读取这个stream并使用response write出去。


我们来简单看下这个过程中一些filter的各自实现。

SimpleHostRoutingFilter这个route类型的filter的shouldFilter方法：

    @Override
    public boolean shouldFilter() {
      // 如果对应的地址是使用host方式，才会生效
      return RequestContext.getCurrentContext().getRouteHost() != null
          && RequestContext.getCurrentContext().sendZuulResponse();
    }

run方法：

    @Override
    public Object run() {
      RequestContext context = RequestContext.getCurrentContext();
      HttpServletRequest request = context.getRequest();
      MultiValueMap<String, String> headers = this.helper
          .buildZuulRequestHeaders(request);
      MultiValueMap<String, String> params = this.helper
          .buildZuulRequestQueryParams(request);
      String verb = getVerb(request);
      InputStream requestEntity = getRequestBody(request);
      if (request.getContentLength() < 0) {
        context.setChunkedRequestBody();
      }

      String uri = this.helper.buildZuulRequestURI(request);
      this.helper.addIgnoredHeaders();

      try {
        HttpResponse response = forward(this.httpClient, verb, uri, request, headers,
            params, requestEntity); // 使用HttpClient调用remoteHost
        setResponse(response); // 设置remoteHost调用的结果
      }
      catch (Exception ex) {
        context.set("error.status_code",
            HttpServletResponse.SC_INTERNAL_SERVER_ERROR);
        context.set("error.exception", ex);
      }
      return null;
    }

    // setResponse方法会把response中的stream放到RequestContext中
    context.setResponseDataStream(entity);

RibbonRoutingFilter跟SimpleHostRoutingFilter类似，区别就是它的shouldFilter方法里不是判断host方式，而是判断路由规则里是否存在serviceId。它的run方法也是使用HttpClient完成服务的调用，但是它是使用ribbon完成的。

访问地址：

    http://localhost:7777/compute-service/add?a=1&b=2

由于使用了serviceId的方式，所以会触发RibbonRoutingFilter并完成服务的调用。

当使用Eureka服务发现的时候，建议使用serviceId的方式，而不是直接host的方式。因为基于serviceId的方式会使用ribbon完成服务的调用，ribbon中又使用了hystrix和loadbalance等功能，有更好的健壮性。


SendResponseFilter是一个post类型的，它会写回服务调用产生的结果。

    @Override
    public boolean shouldFilter() {
      // RibbonRoutingFilter和SimpleHostRoutingFilter都会写入stream数据到RequestContext中的responseDataStream中，所以这个filter会生效
      return !RequestContext.getCurrentContext().getZuulResponseHeaders().isEmpty()
          || RequestContext.getCurrentContext().getResponseDataStream() != null
          || RequestContext.getCurrentContext().getResponseBody() != null;
    }

    @Override
  	public Object run() {
  		try {
  			addResponseHeaders();
        // 最终在RequestContext中使用response write这个stream
  			writeResponse();
  		}
  		catch (Exception ex) {
  			ReflectionUtils.rethrowRuntimeException(ex);
  		}
  		return null;
  	}

SpringCloud默认还加了其它的一些拦截器，有兴趣的读者可以自行查看源代码。


## 总结

Zuul内部的处理使用ZuulServlet完成，ZuulServlet继承HttpServlet，重写了service方法，service方法内部分别是pre、route、post和error类型的filter进行调用。这里的不同类型的filter执行顺序文中已经说明。

要在SpringCloud中使用Zuul，需要加上@EnableZuulProxy注解。加上这个注解之后SpringCloud会构造一些bean，比如ZuulHandlerMapping、DiscoveryClientRouteLocator、各种filter等。其中DiscoveryClientRouteLocator是一个基于服务发现的路由规则生成器，它会基于zuul的配置构造路由规则。ZuulHandlerMapping是一个HandlerMapping的实现，它跟基于路由规则注册handler，其中key为路由规则对应的路径，handler都是ZuulController，ZuulController内部使用ZuulServlet进行请求的处理。

Zuul把真正的服务调用放在了filter中实现。它提供了SimpleHostRoutingFilter和RibbonRoutingFilter这2个route类型的filter用于执行服务。从名字也可以看出来，SimpleHostRoutingFilter用于执行基于host方式的调用url接口，RibbonRoutingFilter基于服务发现的方式调用服务。一般我们都建议使用RibbonRoutingFilter，因为它内部使用ribbon，更加健壮。

## 其它

RoutesEndpoint这个endpoint使用RouteLocator中提供的所有路由规则。

访问：

    http://localhost:7777/routes

得到路由规则：

    {
        /api-a-url/**: "http://localhost:2222/",
        /compute-service/**: "compute-service"
    }


Zuul声称自己可以使用static类型的filter用于处理静态资源，也提供了一个StaticResponseFilter的一个基类，但是查看ZuulServlet的源码发现没有哪段逻辑是处理静态资源的。 上了github发现专门有个[issue](https://github.com/spring-cloud/spring-cloud-netflix/issues/1001)说明目前还不支持自定义的filter。

## 参考资料


https://github.com/Netflix/zuul/

https://github.com/Netflix/zuul/wiki
