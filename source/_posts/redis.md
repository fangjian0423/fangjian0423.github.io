title: Redis简介-安装-入门
date: 2014-08-16 09:49:50
tags:
- cache
- redis
categories:
- cache
description: Redis是完全开源免费的，遵守BSD协议，先进的key - value持久化产品。它通常被称为数据结构服务器 ...

---------------
## 前言 ##

我们team马上要用Redis了。 leader要求学习一下这东西。

Redis大名很早以前就听过了，以前在的公司都没有用到。 现在有机会终于接触到了，果断学习起来。

## 什么是redis ##

[Redis](http://redis.io/)是完全开源免费的，遵守BSD协议，先进的key - value持久化产品。它通常被称为数据结构服务器，因为值（value）可以是 字符串(String), 哈希(Map), 列表(list), 集合(sets)和有序集合(sorted sets)等类型。

当然，我们是通过命令行操作这些数据的。

具体的一些关于命令的东西小伙伴们可以去[http://try.redis.io/](http://try.redis.io/)感受一下。

## redis的安装 ##

Redis在linux下安装比较简单。 略过.....

下面讲下windows下安装Redis。

首先进入[redis下载页面](http://redis.io/download)

![](http://format-blog-image.qiniudn.com/redis1.jpg)

进入之后

![](http://format-blog-image.qiniudn.com/redis2.jpg)

下载的zip解压到指定的目录。

/redis/bin/release目录下结构有个压缩包，直接解压。 目录内文件如下：

![](http://format-blog-image.qiniudn.com/redis3.jpg)

redis-server.exe 表示服务端程序。
redis-cli.exe    表示客户端程序。

先启动redis服务器：

![](http://format-blog-image.qiniudn.com/redis4.jpg)

这里注意一下，启动服务器的时候需要配置文件，直接在命令行后面加上配置文件的路径即可。

命令行最后  "The server is now ready to accept connections on port 6397"  也说明了服务器启动成功。

接下来启动客户端：

![](http://format-blog-image.qiniudn.com/redis5.jpg)

ok, 安装成功。

## Java操作Redis ##

maven加入redis依赖。

	<dependency>
		<groupId>redis.clients</groupId>
		<artifactId>jedis</artifactId>
		<version>2.5.1</version>
	</dependency>

Java：

	import org.junit.Before;
	import org.junit.Test;
	import redis.clients.jedis.Jedis;
	import redis.clients.jedis.JedisPool;
	import redis.clients.jedis.JedisPoolConfig;
	
	import java.util.Set;
	
	public class RedisTest {
	
	    private JedisPool pool;
	    private Jedis jedis;
	
	    @Before
	    public void setUp() {
	        this.pool = new JedisPool(new JedisPoolConfig(), "127.0.0.1");
	        this.jedis = pool.getResource();
	    }
	
	    @Test
	    public void testGetName() {
	        System.out.println(jedis.get("name"));
	    }
	
	    @Test
	    public void testDel() {
	        jedis.set("age", "99");
	        System.out.println(jedis.get("age"));
	        jedis.del("age");
	        System.out.println(jedis.get("age"));
	    }
	
	    @Test
	    public void testKeys() {
	        Set<String> keys = jedis.keys("*");
	        System.out.println(keys);
	    }
	
	}

简单地测试了几个方法。 其他方法名跟redis命令基本类似，所以还是得熟悉redis命令。

## 总结 ##

简单地安装了一下redis，然后用Java访问了Redis服务器，并操作了一些数据。

接下来就是熟悉[redis的各种命令](http://redis.io/commands)了。  go go go!~
