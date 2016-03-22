title: Spring与Mybatis整合的MapperScannerConfigurer处理过程源码分析
date: 2014-09-06 21:55:12
tags:
- mybatis
- spring
categories: 
- mybatis
description: 本文将分析mybatis与spring整合的MapperScannerConfigurer的底层原理 ...

---

## 前言 ##
本文将分析mybatis与spring整合的MapperScannerConfigurer的底层原理，之前已经分析过java中实现动态，可以使用jdk自带api和cglib第三方库生成动态代理。本文分析的mybatis版本3.2.7，mybatis-spring版本1.2.2。

## MapperScannerConfigurer介绍 ##
[MapperScannerConfigurer](https://mybatis.github.io/spring/zh/mappers.html#MapperScannerConfigurer)是spring和mybatis整合的mybatis-spring jar包中提供的一个类。 

想要了解该类的作用，就得先了解[MapperFactoryBean](https://mybatis.github.io/spring/zh/mappers.html#MapperFactoryBean)。

MapperFactoryBean的出现为了代替手工使用SqlSessionDaoSupport或SqlSessionTemplate编写数据访问对象(DAO)的代码,使用动态代理实现。

比如下面这个官方文档中的配置：

    <bean id="userMapper" class="org.mybatis.spring.mapper.MapperFactoryBean">
      <property name="mapperInterface" value="org.mybatis.spring.sample.mapper.UserMapper" />
      <property name="sqlSessionFactory" ref="sqlSessionFactory" />
    </bean>
    
org.mybatis.spring.sample.mapper.UserMapper是一个接口，我们创建一个MapperFactoryBean实例，然后注入这个接口和sqlSessionFactory（mybatis中提供的SqlSessionFactory接口，MapperFactoryBean会使用SqlSessionFactory创建SqlSession）这两个属性。

之后想使用这个UserMapper接口的话，直接通过spring注入这个bean，然后就可以直接使用了，spring内部会创建一个这个接口的动态代理。


当发现要使用多个MapperFactoryBean的时候，一个一个定义肯定非常麻烦，于是mybatis-spring提供了MapperScannerConfigurer这个类，它将会查找类路径下的映射器并自动将它们创建成MapperFactoryBean。

    <bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
      <property name="basePackage" value="org.mybatis.spring.sample.mapper" />
    </bean>
    
这段配置会扫描org.mybatis.spring.sample.mapper下的所有接口，然后创建各自接口的动态代理类。

## MapperScannerConfigurer底层代码分析 ##
以以下代码为示例进行讲解(部分代码，其他代码及配置省略)：

    package org.format.dynamicproxy.mybatis.dao;
    public interface UserDao {
        public User getById(int id);
        public int add(User user);    
        public int update(User user);    
        public int delete(User user);    
        public List<User> getAll();    
    }
    
    <bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
        <property name="basePackage" value="org.format.dynamicproxy.mybatis.dao"/>
        <property name="sqlSessionFactoryBeanName" value="sqlSessionFactory"/>
    </bean>
    
    
我们先通过测试用例debug查看userDao的实现类到底是什么。
![](http://format-blog-image.qiniudn.com/mybatis1.png)
我们可以看到，userDao是1个MapperProxy类的实例。
看下MapperProxy的源码，没错，实现了InvocationHandler，说明使用了jdk自带的动态代理。

    public class MapperProxy<T> implements InvocationHandler, Serializable {
    
      private static final long serialVersionUID = -6424540398559729838L;
      private final SqlSession sqlSession;
      private final Class<T> mapperInterface;
      private final Map<Method, MapperMethod> methodCache;
    
      public MapperProxy(SqlSession sqlSession, Class<T> mapperInterface, Map<Method, MapperMethod> methodCache) {
        this.sqlSession = sqlSession;
        this.mapperInterface = mapperInterface;
        this.methodCache = methodCache;
      }
    
      public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        if (Object.class.equals(method.getDeclaringClass())) {
          try {
            return method.invoke(this, args);
          } catch (Throwable t) {
            throw ExceptionUtil.unwrapThrowable(t);
          }
        }
        final MapperMethod mapperMethod = cachedMapperMethod(method);
        return mapperMethod.execute(sqlSession, args);
      }
    
      private MapperMethod cachedMapperMethod(Method method) {
        MapperMethod mapperMethod = methodCache.get(method);
        if (mapperMethod == null) {
          mapperMethod = new MapperMethod(mapperInterface, method, sqlSession.getConfiguration());
          methodCache.put(method, mapperMethod);
        }
        return mapperMethod;
      }
    
    }

### 下面开始分析MapperScannerConfigurer的源码 ###

MapperScannerConfigurer实现了BeanDefinitionRegistryPostProcessor接口，BeanDefinitionRegistryPostProcessor接口是一个可以修改spring工长中已定义的bean的接口，该接口有个postProcessBeanDefinitionRegistry方法。
![](http://format-blog-image.qiniudn.com/mybatis2.png)

然后我们看下ClassPathMapperScanner中的关键是如何扫描对应package下的接口的。
![](http://format-blog-image.qiniudn.com/mybatis3.png)

其实MapperScannerConfigurer的作用也就是将对应的接口的类型改造为MapperFactoryBean，而这个MapperFactoryBean的属性mapperInterface是原类型。MapperFactoryBean本文开头已分析过。

所以最终我们还是要分析MapperFactoryBean的实现原理！

MapperFactoryBean继承了SqlSessionDaoSupport类，SqlSessionDaoSupport类继承DaoSupport抽象类，DaoSupport抽象类实现了InitializingBean接口，因此实例个MapperFactoryBean的时候，都会调用InitializingBean接口的afterPropertiesSet方法。

DaoSupport的afterPropertiesSet方法：
![](http://format-blog-image.qiniudn.com/mybatis4.png)
MapperFactoryBean重写了checkDaoConfig方法：
![](http://format-blog-image.qiniudn.com/mybatis5.png)
然后通过spring工厂拿对应的bean的时候：
![](http://format-blog-image.qiniudn.com/mybatis6.png)
这里的SqlSession是SqlSessionTemplate，SqlSessionTemplate的getMapper方法：
![](http://format-blog-image.qiniudn.com/mybatis7.png)
Configuration的getMapper方法，会使用MapperRegistry的getMapper方法：
![](http://format-blog-image.qiniudn.com/mybatis8.png)
MapperRegistry的getMapper方法：
![](http://format-blog-image.qiniudn.com/mybatis9.png)
MapperProxyFactory构造MapperProxy：
![](http://format-blog-image.qiniudn.com/mybatis10.png)
**没错！ MapperProxyFactory就是使用了jdk组带的Proxy完成动态代理。**
MapperProxy本来一开始已经提到。MapperProxy内部使用了MapperMethod类完成方法的调用：
![](http://format-blog-image.qiniudn.com/mybatis11.png)

下面，我们以UserDao的getById方法来debug看看MapperMethod的execute方法是如何走的。

    @Test
    public void testGet() {
        int id = 1;
        System.out.println(userDao.getById(id));
    }
    <select id="getById" parameterType="int" resultType="org.format.dynamicproxy.mybatis.bean.User">
        SELECT * FROM users WHERE id = #{id}
    </select>
    
![](http://format-blog-image.qiniudn.com/mybatis12.png)
![](http://format-blog-image.qiniudn.com/mybatis13.png)

示例代码：[https://github.com/fangjian0423/dynamic-proxy-mybatis-study](https://github.com/fangjian0423/dynamic-proxy-mybatis-study)

## 总结 ##
来到了新公司，接触了Mybatis，以前接触过～ 但是接触的不深入，突然发现spring与mybatis整合之后可以只写个接口而不实现，spring默认会帮我们实现，然后觉得非常神奇，于是写了篇[java动态代码浅析](http://www.cnblogs.com/fangjian0423/p/java-dynamic-proxy.html)和本文。
## 参考资料 ##
[https://mybatis.github.io/spring/zh/mappers.html](https://mybatis.github.io/spring/zh/mappers.html)


