title: Loadbalance的几种算法以及在ribbon中的使用
date: 2017-01-29 19:32:16
tags:
- springcloud
- algorithm
categories: springcloud

----------------


Load Balance负载均衡是用于解决一台机器(一个进程)无法解决所有请求而产生的一种算法。

像nginx可以使用负载均衡分配流量，ribbon为客户端提供负载均衡，dubbo服务调用里的负载均衡等等，很多地方都使用到了负载均衡。

使用负载均衡带来的好处很明显：

1. 当集群里的1台或者多台服务器down的时候，剩余的没有down的服务器可以保证服务的继续使用
2. 使用了更多的机器保证了机器的良性使用，不会由于某一高峰时刻导致系统cpu急剧上升

负载均衡有好几种实现策略，常见的有：

1. 随机 (Random)
2. 轮询 (RoundRobin)
3. 一致性哈希 (ConsistentHash)
4. 哈希 (Hash)
5. 加权（Weighted）

<!--more-->

我们以[ribbon](https://github.com/Netflix/ribbon)的实现为基础，看看其中的一些算法是如何实现的。

ribbon是一个为客户端提供负载均衡功能的服务，它内部提供了一个叫做ILoadBalance的接口代表负载均衡器的操作，比如有添加服务器操作、选择服务器操作、获取所有的服务器列表、获取可用的服务器列表等等。

还提供了一个叫做IRule的接口代表负载均衡策略：

	public interface IRule{
    	public Server choose(Object key);
    	public void setLoadBalancer(ILoadBalancer lb);
    	public ILoadBalancer getLoadBalancer();    
	}
	
IRule接口的实现类有以下几种：

![image](http://7x2wh6.com1.z0.glb.clouddn.com/loadbalance01.png)

其中RandomRule表示随机策略、RoundRobin表示轮询策略、WeightedResponseTimeRule表示加权策略、BestAvailableRule表示请求数最少策略等等。

随机策略很简单，就是从服务器中随机选择一个服务器，RandomRule的实现代码如下：

	public Server choose(ILoadBalancer lb, Object key) {
        if (lb == null) {
            return null;
        }
        Server server = null;

        while (server == null) {
            if (Thread.interrupted()) {
                return null;
            }
            List<Server> upList = lb.getReachableServers();
            List<Server> allList = lb.getAllServers();
            int serverCount = allList.size();
            if (serverCount == 0) {
                return null;
            }
            int index = rand.nextInt(serverCount); // 使用jdk内部的Random类随机获取索引值index
            server = upList.get(index); // 得到服务器实例

            if (server == null) {
                Thread.yield();
                continue;
            }

            if (server.isAlive()) {
                return (server);
            }

            server = null;
            Thread.yield();
        }
        return server;
    }
    
RoundRobin轮询策略表示每次都取下一个服务器，比如一共有5台服务器，第1次取第1台，第2次取第2台，第3次取第3台，以此类推：

	public Server choose(ILoadBalancer lb, Object key) {
        if (lb == null) {
            log.warn("no load balancer");
            return null;
        }

        Server server = null;
        int count = 0;
        while (server == null && count++ < 10) { // retry 10 次
            List<Server> reachableServers = lb.getReachableServers();
            List<Server> allServers = lb.getAllServers();
            int upCount = reachableServers.size();
            int serverCount = allServers.size();

            if ((upCount == 0) || (serverCount == 0)) {
                log.warn("No up servers available from load balancer: " + lb);
                return null;
            }

            int nextServerIndex = incrementAndGetModulo(serverCount); // incrementAndGetModulo方法内部使用nextServerCyclicCounter这个AtomicInteger属性原子递增对serverCount取模得到索引值
            server = allServers.get(nextServerIndex); // 得到服务器实例

            if (server == null) {
                Thread.yield();
                continue;
            }

            if (server.isAlive() && (server.isReadyToServe())) {
                return (server);
            }

            server = null;
        }

        if (count >= 10) {
            log.warn("No available alive servers after 10 tries from load balancer: "
                    + lb);
        }
        return server;
    }
	
BestAvailableRule策略用来选取最少并发量请求的服务器：

	public Server choose(Object key) {
        if (loadBalancerStats == null) {
            return super.choose(key);
        }
        List<Server> serverList = getLoadBalancer().getAllServers(); // 获取所有的服务器列表
        int minimalConcurrentConnections = Integer.MAX_VALUE;
        long currentTime = System.currentTimeMillis();
        Server chosen = null;
        for (Server server: serverList) { // 遍历每个服务器
            ServerStats serverStats = loadBalancerStats.getSingleServerStat(server); // 获取各个服务器的状态
            if (!serverStats.isCircuitBreakerTripped(currentTime)) { // 没有触发断路器的话继续执行
                int concurrentConnections = serverStats.getActiveRequestsCount(currentTime); // 获取当前服务器的请求个数
                if (concurrentConnections < minimalConcurrentConnections) { // 比较各个服务器之间的请求数，然后选取请求数最少的服务器并放到chosen变量中
                    minimalConcurrentConnections = concurrentConnections;
                    chosen = server;
                }
            }
        }
        if (chosen == null) { // 如果没有选上，调用父类ClientConfigEnabledRoundRobinRule的choose方法，也就是使用RoundRobinRule轮询的方式进行负载均衡        
            return super.choose(key);
        } else {
            return chosen;
        }
    }


## 实例验证Ribbon中的LoadBalance功能

ServerList中提供了3个instance，分别是：

	compute-service:2222
	compute-service:2223
	compute-service:2224
	
然后使用不同的IRule策略查看负载均衡的实现。

首先先使用ribbon提供的LoadBalanced注解加在RestTemplate上面，这个注解会自动构造LoadBalancerClient接口的实现类并注册到Spring容器中。

	@Bean
    @LoadBalanced
    RestTemplate restTemplate() {
        return new RestTemplate();
    }

接下来使用RestTemplate进行rest操作的时候，会自动使用负载均衡策略，它内部会在RestTemplate中加入LoadBalancerInterceptor这个拦截器，这个拦截器的作用就是使用负载均衡。

例子中，我们的实例的name叫做compute-service，里面提供了一个方法add用于相加2个Integer类型的数值。


loadbalance的具体操作：

	public String loadbalance() {
        ServiceInstance serviceInstance = loadBalancerClient.choose("compute-service");
        StringBuilder sb = new StringBuilder();
        sb.append("host: ").append(serviceInstance.getHost()).append(", ");
        sb.append("port: ").append(serviceInstance.getPort()).append(", ");
        sb.append("uri: ").append(serviceInstance.getUri());
        return sb.toString();
    }

### RandomRule随机策略

RandomRule：
	
	@Configuration
	public class RibbonConfiguration {

    	@Autowired
	    private SpringClientFactory springClientFactory;

    	@Bean
	    public IRule ribbonRule() {
			return new RandomRule();
    	}

	}

测试结果如下，确实是随机获取的：

	host: 192.168.31.113, port: 2222, uri: http://192.168.31.113:2222
	host: 192.168.31.113, port: 2222, uri: http://192.168.31.113:2222
	host: 192.168.31.113, port: 2222, uri: http://192.168.31.113:2222
	host: 192.168.31.113, port: 2224, uri: http://192.168.31.113:2224
	host: 192.168.31.113, port: 2224, uri: http://192.168.31.113:2224
	host: 192.168.31.113, port: 2222, uri: http://192.168.31.113:2222
	host: 192.168.31.113, port: 2223, uri: http://192.168.31.113:2223
	host: 192.168.31.113, port: 2222, uri: http://192.168.31.113:2222
	host: 192.168.31.113, port: 2222, uri: http://192.168.31.113:2222
	host: 192.168.31.113, port: 2223, uri: http://192.168.31.113:2223	
### RoundRobinRule轮询策略
	
RoundRobinRule：

	@Bean
	public IRule ribbonRule() {
		return new RandomRule();
    }

测试结果如下，确实是轮询每个服务器的：

	host: 192.168.31.113, port: 2223, uri: http://192.168.31.113:2223
	host: 192.168.31.113, port: 2222, uri: http://192.168.31.113:2222
	host: 192.168.31.113, port: 2224, uri: http://192.168.31.113:2224
	host: 192.168.31.113, port: 2223, uri: http://192.168.31.113:2223
	host: 192.168.31.113, port: 2222, uri: http://192.168.31.113:2222
	host: 192.168.31.113, port: 2224, uri: http://192.168.31.113:2224
	host: 192.168.31.113, port: 2223, uri: http://192.168.31.113:2223
	host: 192.168.31.113, port: 2222, uri: http://192.168.31.113:2222
	host: 192.168.31.113, port: 2224, uri: http://192.168.31.113:2224
	host: 192.168.31.113, port: 2223, uri: http://192.168.31.113:2223
	host: 192.168.31.113, port: 2222, uri: http://192.168.31.113:2222
	host: 192.168.31.113, port: 2224, uri: http://192.168.31.113:2224
	
### BestAvailableRule最少并发数策略
	
BestAvailableRule：

	@Bean
	public IRule ribbonRule() {
		return new BestAvailableRule();
    }

如果直接访问浏览器的话，测试结果如下(因为每次访问完请求数都变成0，下次遍历永远都是2223这个端口的实例)：

	host: 192.168.31.113, port: 2223, uri: http://192.168.31.113:2223
	host: 192.168.31.113, port: 2223, uri: http://192.168.31.113:2223
	host: 192.168.31.113, port: 2223, uri: http://192.168.31.113:2223
	host: 192.168.31.113, port: 2223, uri: http://192.168.31.113:2223
	host: 192.168.31.113, port: 2223, uri: http://192.168.31.113:2223
	host: 192.168.31.113, port: 2223, uri: http://192.168.31.113:2223
	..
	
使用wrk模拟并发请求，结果会出现多个实例：

	wrk -c 1000 -t 10 -d 10s http://localhost:3333/test
	
