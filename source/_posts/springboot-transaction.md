title: SpringBoot的事务管理
date: 2016-10-07 01:28:36
tags:
- springboot
categories: springboot

----------------

Springboot内部提供的事务管理器是根据autoconfigure来进行决定的。

比如当使用jpa的时候，也就是pom中加入了spring-boot-starter-data-jpa这个starter之后(之前我们分析过[springboot的自动化配置原理](http://fangjian0423.github.io/2016/06/12/springboot-autoconfig-analysis/))。

Springboot会构造一个JpaTransactionManager这个事务管理器。

而当我们使用spring-boot-starter-jdbc的时候，构造的事务管理器则是DataSourceTransactionManager。

这2个事务管理器都实现了spring中提供的PlatformTransactionManager接口，这个接口是spring的事务核心接口。

这个核心接口有以下这几个常用的实现策略：

1. HibernateTransactionManager
2. DataSourceTransactionManager
3. JtaTransactionManager
4. JpaTransactionManager

具体的PlatformTransactionManager继承关系如下：

![](http://7x2wh6.com1.z0.glb.clouddn.com/transactionmanager.png)
	

<!--more-->
	
spring-boot-starter-data-jpa这个starter会触发HibernateJpaAutoConfiguration这个自动化配置类，HibernateJpaAutoConfiguration继承了JpaBaseConfiguration基础类。

在JpaBaseConfiguration中构造了事务管理器：

	@Bean
	@ConditionalOnMissingBean(PlatformTransactionManager.class)
	public PlatformTransactionManager transactionManager() {
		return new JpaTransactionManager();
	}
	
spring-boot-starter-jdbc会触发DataSourceTransactionManagerAutoConfiguration这个自动化配置类，也会构造事务管理器：

	@Bean
	@ConditionalOnMissingBean(PlatformTransactionManager.class)
	@ConditionalOnBean(DataSource.class)
	public DataSourceTransactionManager transactionManager() {
		return new DataSourceTransactionManager(this.dataSource);
	}
	

Spring的事务管理器PlatformTransactionManager接口中定义了3个方法：


	// 基于事务的传播特性，返回一个已经存在的事务或者创建一个新的事务
	TransactionStatus getTransaction(TransactionDefinition definition) throws TransactionException;
	
	// 提交事务
	void commit(TransactionStatus status) throws TransactionException;
	
	// 回滚事务
	void rollback(TransactionStatus status) throws TransactionException;
	
其中TransactionDefinition接口表示跟spring兼容的事务属性，比如传播行为、隔离级别、超时时间、是否只读等属性。

DefaultTransactionDefinition类是一个默认的TransactionDefinition实现，它的传播行为是PROPAGATION_REQUIRED(如果当前没事务，则创建一个，否则加入到当前事务中)，隔离级别是数据库默认级别。

TransactionStatus接口表示事务的状态，比如事务是否是一个刚构造的事务、事务是否已经完成等状态。

下面这段代码就是传统事务的常见写法：


	transaction.begin();
	try {
		...
		transaction.commit();
	} catch(Exception e) {
		...
		transaction.rollback();
	} finally {
		
	}


由于spring的事务操作被封装到了PlatformTransactionManager接口中，commit和rollback方法对应接口中的方法，begin方法在getTransaction方法中会被调用。


细心的读者发现文章前面构造事务管理器的时候都会加上这段注解：

	@ConditionalOnMissingBean(PlatformTransactionManager.class)
	
也就是说如果我们手动配置了事务管理器，Springboot就不会再为我们自动配置事务管理器。

如果要使用多个事务管理器的话，那么需要手动配置多个：

	@Configuration
    public class DatabaseConfiguration {
    
        @Bean
        public PlatformTransactionManager transactionManager1(EntityManagerFactory entityManagerFactory) {
            return new JpaTransactionManager(entityManagerFactory);
        }
    
        @Bean
        public PlatformTransactionManager transactionManager2(DataSource dataSource) {
            return new DataSourceTransactionManager(dataSource);
        }
    
    }

然后使用Transactional注解的时候需要声明是哪个事务管理器：

	@Transactional(value="transactionManager1")
    public void save() {
        doSave();
    }


Spring给我们提供了一个TransactionManagementConfigurer接口，该接口只有一个方法返回PlatformTransactionManager。其中返回的PlatformTransactionManager就表示这是默认的事务处理器，这样在Transactional注解上就不需要声明是使用哪个事务管理器了。



参考资料：

http://www.cnblogs.com/davidwang456/p/4309038.html

http://blog.csdn.net/chjttony/article/details/6528344


