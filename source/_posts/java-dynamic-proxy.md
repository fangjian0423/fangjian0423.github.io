title: java动态代理浅析
date: 2014-08-16 15:49:50
tags: [java,proxy,cglib]
description: 记录java动态代理的一些实现，包括jdk自带的动态代理和第三方库cglib

-------------

最近在公司看到了mybatis与spring整合中MapperScannerConfigurer的使用，该类通过反向代理自动生成基于接口的动态代理类。

于是想起了java的动态代理，然后就有了这篇文章。

本文使用动态代理模拟处理事务的拦截器。

接口：

    public interface UserService {
        public void addUser();
        public void removeUser();
        public void searchUser();
    }
    
实现类：    

    public class UserServiceImpl implements UserService {
        public void addUser() {
            System.out.println("add user");
        }
        public void removeUser() {
            System.out.println("remove user");
        }
        public void searchUser() {
            System.out.println("search user");
        }
    }

## java动态代理的实现有2种方式 ##

### 1.jdk自带的动态代理 ###

使用jdk自带的动态代理需要了解InvocationHandler接口和Proxy类，他们都是在java.lang.reflect包下。

InvocationHandler介绍：

InvocationHandler是代理实例的调用处理程序实现的接口。

每个代理实例都具有一个关联的InvocationHandler。对代理实例调用方法时，这个方法会调用InvocationHandler的invoke方法。

Proxy介绍：

Proxy 提供静态方法用于创建动态代理类和实例。

实例(模拟AOP处理事务)：

    public class TransactionInterceptor implements InvocationHandler {

        private Object target;
    
        public void setTarget(Object target) {
            this.target = target;
        }
        
        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            System.out.println("start Transaction");
	        method.invoke(target, args);
	        System.out.println("end Transaction");
            return null;
        }

    }
    
测试代码：

    public class TestDynamicProxy {

	    @Test
	    public void testJDK() {
	        TransactionInterceptor transactionInterceptor = new TransactionInterceptor();
	        UserService userService = new UserServiceImpl();
	        transactionInterceptor.setTarget(userService);
	        UserService userServiceProxy =
	                (UserService) Proxy.newProxyInstance(
	                        userService.getClass().getClassLoader(),
	                        userService.getClass().getInterfaces(),
	                        transactionInterceptor);
	        userServiceProxy.addUser();
	    }

	}
    
测试结果：

    start Transaction
	add user
	end Transaction

很明显，我们通过userServiceProxy这个代理类进行方法调用的时候，会在方法调用前后进行事务的开启和关闭。

### 2. 第三方库cglib ###

CGLIB是一个功能强大的，高性能、高质量的代码生成库，用于在运行期扩展Java类和实现Java接口。 

它与JDK的动态代理的之间最大的区别就是：

**JDK动态代理是针对接口的，而cglib是针对类来实现代理的，cglib的原理是对指定的目标类生成一个子类，并覆盖其中方法实现增强，但因为采用的是继承，所以不能对final修饰的类进行代理。**

实例：

	public class UserServiceCallBack implements MethodInterceptor {
	
	    @Override
	    public Object intercept(Object o, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
	        System.out.println("start Transaction by cglib");
	        methodProxy.invokeSuper(o, args);
	        System.out.println("end Transaction by cglib");
	        return null;
	    }
	
	}

测试代码：

	public class TestDynamicProxy {
	
	    @Test
	    public void testCGLIB() {
	        Enhancer enhancer = new Enhancer();
	        enhancer.setSuperclass(UserServiceImpl.class);
	        enhancer.setCallback(new UserServiceCallBack());
	        UserServiceImpl proxy = (UserServiceImpl)enhancer.create();
	        proxy.addUser();
	    }
	
	}

测试结果：

	start Transaction by cglib
	add user
	end Transaction by cglib

## 结束语 ##

简单讲解了JDK和cglib这2个动态代理，之后会再写篇文章讲讲MapperScannerConfigurer的原理实现。
