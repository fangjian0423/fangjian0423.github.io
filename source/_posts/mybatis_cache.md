title: 通过源码分析MyBatis的缓存
date: 2014-12-10 14:34:05
tags: [MyBatis,cache]
description: 介绍MyBatis中缓存的概念，并通过源码进行分析
----------------

前方高能！ 本文内容有点多，通过实际测试例子+源码分析的方式解剖MyBatis缓存的概念，对这方面有兴趣的小伙伴请继续看下去~

## MyBatis缓存介绍 ##
首先看一段[wiki](http://zh.wikipedia.org/wiki/MyBatis)上关于MyBatis缓存的介绍：

MyBatis支持声明式数据缓存（declarative data caching）。当一条SQL语句被标记为“可缓存”后，首次执行它时从数据库获取的所有数据会被存储在一段高速缓存中，今后执行这条语句时就会从高速缓存中读取结果，而不是再次命中数据库。MyBatis提供了默认下基于Java HashMap的缓存实现，以及用于与OSCache、Ehcache、Hazelcast和Memcached连接的默认连接器。MyBatis还提供API供其他缓存实现使用。

重点的那句话就是：**MyBatis执行SQL语句之后，这条语句就是被缓存，以后再执行这条语句的时候，会直接从缓存中拿结果，而不是再次执行SQL**

这也就是大家常说的MyBatis一级缓存，一级缓存的作用域scope是SqlSession。

MyBatis同时还提供了一种全局作用域global scope的缓存，这也叫做二级缓存，也称作全局缓存。

## 一级缓存 ##

### 测试 ###

同个session进行两次相同查询：

	@Test
    public void test() {
        SqlSession sqlSession = sqlSessionFactory.openSession();
        try {
            User user = (User)sqlSession.selectOne("org.format.mybatis.cache.UserMapper.getById", 1);
            log.debug(user);
            User user2 = (User)sqlSession.selectOne("org.format.mybatis.cache.UserMapper.getById", 1);
            log.debug(user2);
        } finally {
            sqlSession.close();
        }
    }
    
MyBatis只进行1次数据库查询：

    ==>  Preparing: select * from USERS WHERE ID = ? 
    ==> Parameters: 1(Integer)
    <==      Total: 1
    User{id=1, name='format', age=23, birthday=Sun Oct 12 23:20:13 CST 2014}
    User{id=1, name='format', age=23, birthday=Sun Oct 12 23:20:13 CST 2014}

同个session进行两次不同的查询：

	@Test
    public void test() {
        SqlSession sqlSession = sqlSessionFactory.openSession();
        try {
            User user = (User)sqlSession.selectOne("org.format.mybatis.cache.UserMapper.getById", 1);
            log.debug(user);
            User user2 = (User)sqlSession.selectOne("org.format.mybatis.cache.UserMapper.getById", 2);
            log.debug(user2);
        } finally {
            sqlSession.close();
        }
    }

MyBatis进行两次数据库查询：

    ==>  Preparing: select * from USERS WHERE ID = ? 
    ==> Parameters: 1(Integer)
    <==      Total: 1
    User{id=1, name='format', age=23, birthday=Sun Oct 12 23:20:13 CST 2014}
    ==>  Preparing: select * from USERS WHERE ID = ? 
    ==> Parameters: 2(Integer)
    <==      Total: 1
    User{id=2, name='FFF', age=50, birthday=Sat Dec 06 17:12:01 CST 2014}

不同session，进行相同查询： 

	@Test
    public void test() {
        SqlSession sqlSession = sqlSessionFactory.openSession();
        SqlSession sqlSession2 = sqlSessionFactory.openSession();
        try {
            User user = (User)sqlSession.selectOne("org.format.mybatis.cache.UserMapper.getById", 1);
            log.debug(user);
            User user2 = (User)sqlSession2.selectOne("org.format.mybatis.cache.UserMapper.getById", 1);
            log.debug(user2);
        } finally {
            sqlSession.close();
            sqlSession2.close();
        }
    }

MyBatis进行了两次数据库查询：

    ==>  Preparing: select * from USERS WHERE ID = ? 
    ==> Parameters: 1(Integer)
    <==      Total: 1
    User{id=1, name='format', age=23, birthday=Sun Oct 12 23:20:13 CST 2014}
    ==>  Preparing: select * from USERS WHERE ID = ? 
    ==> Parameters: 1(Integer)
    <==      Total: 1
    User{id=1, name='format', age=23, birthday=Sun Oct 12 23:20:13 CST 2014}

同个session,查询之后更新数据，再次查询相同的语句：

	@Test
    public void test() {
        SqlSession sqlSession = sqlSessionFactory.openSession();
        try {
            User user = (User)sqlSession.selectOne("org.format.mybatis.cache.UserMapper.getById", 1);
            log.debug(user);
            user.setAge(100);
            sqlSession.update("org.format.mybatis.cache.UserMapper.update", user);
            User user2 = (User)sqlSession.selectOne("org.format.mybatis.cache.UserMapper.getById", 1);
            log.debug(user2);
            sqlSession.commit();
        } finally {
            sqlSession.close();
        }
    }
    
更新操作之后缓存会被清除：

	==>  Preparing: select * from USERS WHERE ID = ? 
    ==> Parameters: 1(Integer)
    <==      Total: 1
    User{id=1, name='format', age=23, birthday=Sun Oct 12 23:20:13 CST 2014}
    ==>  Preparing: update USERS SET NAME = ? , AGE = ? , BIRTHDAY = ? where ID = ? 
    ==> Parameters: format(String), 23(Integer), 2014-10-12 23:20:13.0(Timestamp), 1(Integer)
    <==    Updates: 1
    ==>  Preparing: select * from USERS WHERE ID = ? 
    ==> Parameters: 1(Integer)
    <==      Total: 1
    User{id=1, name='format', age=23, birthday=Sun Oct 12 23:20:13 CST 2014}


很明显，结果验证了一级缓存的概念，**在同个SqlSession中，查询语句相同的sql会被缓存，但是一旦执行新增或更新或删除操作，缓存就会被清除**

### 源码分析 ###

在分析MyBatis的一级缓存之前，我们先简单看下MyBatis中几个重要的类和接口：

org.apache.ibatis.session.Configuration类：MyBatis全局配置信息类

org.apache.ibatis.session.SqlSessionFactory接口：操作SqlSession的工厂接口，具体的实现类是DefaultSqlSessionFactory

org.apache.ibatis.session.SqlSession接口：执行sql，管理事务的接口，具体的实现类是DefaultSqlSession

org.apache.ibatis.executor.Executor接口：sql执行器，SqlSession执行sql最终是通过该接口实现的，常用的实现类有SimpleExecutor和CachingExecutor,这些实现类都使用了[装饰者设计模式](http://zh.wikipedia.org/wiki/%E4%BF%AE%E9%A5%B0%E6%A8%A1%E5%BC%8F)

	
一级缓存的作用域是SqlSession，那么我们就先看一下SqlSession的select过程：

这是DefaultSqlSession（SqlSession接口实现类，MyBatis默认使用这个类）的selectList源码（我们例子上使用的是selectOne方法，调用selectOne方法最终会执行selectList方法）：

	public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
	    try {
	      MappedStatement ms = configuration.getMappedStatement(statement);
	      List<E> result = executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
	      return result;
	    } catch (Exception e) {
	      throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
	    } finally {
	      ErrorContext.instance().reset();
	    }
    }

我们看到SqlSession最终会调用Executor接口的方法。

接下来我们看下DefaultSqlSession中的executor接口属性具体是哪个实现类。

DefaultSqlSession的构造过程（DefaultSqlSessionFactory内部）：

	private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit) {
        Transaction tx = null;
        try {
          final Environment environment = configuration.getEnvironment();
          final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
          tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
          final Executor executor = configuration.newExecutor(tx, execType, autoCommit);
          return new DefaultSqlSession(configuration, executor);
        } catch (Exception e) {
          closeTransaction(tx); // may have fetched a connection so lets call close()
          throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
        } finally {
          ErrorContext.instance().reset();
        }
	}

我们看到DefaultSqlSessionFactory构造DefaultSqlSession的时候，Executor接口的实现类是由Configuration构造的：

	public Executor newExecutor(Transaction transaction, ExecutorType executorType, boolean autoCommit) {
        executorType = executorType == null ? defaultExecutorType : executorType;
        executorType = executorType == null ? ExecutorType.SIMPLE : executorType;
        Executor executor;
        if (ExecutorType.BATCH == executorType) {
          executor = new BatchExecutor(this, transaction);
        } else if (ExecutorType.REUSE == executorType) {
          executor = new ReuseExecutor(this, transaction);
        } else {
          executor = new SimpleExecutor(this, transaction);
        }
        if (cacheEnabled) {
          executor = new CachingExecutor(executor, autoCommit);
        }
        executor = (Executor) interceptorChain.pluginAll(executor);
        return executor;
	}

Executor根据ExecutorType的不同而创建，最常用的是SimpleExecutor，本文的例子也是创建这个实现类。 最后我们发现如果cacheEnabled这个属性为true的话，那么executor会被包一层装饰器，这个装饰器是CachingExecutor。其中cacheEnabled这个属性是mybatis总配置文件中settings节点中cacheEnabled子节点的值，默认就是true，也就是说我们在mybatis总配置文件中不配cacheEnabled的话，它也是默认为打开的。

现在，问题就剩下一个了，**CachingExecutor执行sql的时候到底做了什么？**

带着这个问题，我们继续走下去（CachingExecutor的query方法）：
    
    public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
        Cache cache = ms.getCache();
        if (cache != null) {
          flushCacheIfRequired(ms);
          if (ms.isUseCache() && resultHandler == null) { 
            ensureNoOutParams(ms, parameterObject, boundSql);
            if (!dirty) {
              cache.getReadWriteLock().readLock().lock();
              try {
                @SuppressWarnings("unchecked")
                List<E> cachedList = (List<E>) cache.getObject(key);
                if (cachedList != null) return cachedList;
              } finally {
                cache.getReadWriteLock().readLock().unlock();
              }
            }
            List<E> list = delegate.<E> query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
            tcm.putObject(cache, key, list); // issue #578. Query must be not synchronized to prevent deadlocks
            return list;
          }
        }
        return delegate.<E>query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
    }
        
其中Cache cache = ms.getCache();这句代码中，这个cache实际上就是个二级缓存，由于我们没有开启二级缓存(二级缓存的内容下面会分析)，因此这里执行了最后一句话。这里的delegate也就是SimpleExecutor,SimpleExecutor没有Override父类的query方法，因此最终执行了SimpleExecutor的父类BaseExecutor的query方法。
    
**所以一级缓存最重要的代码就是BaseExecutor的query方法!**

![](http://images.cnblogs.com/cnblogs_com/fangjian0423/603237/o_mybatis_cache01.jpg)	

**BaseExecutor的属性localCache是个PerpetualCache类型的实例，PerpetualCache类是实现了MyBatis的Cache缓存接口的实现类之一，内部有个Map<Object, Object>类型的属性用来存储缓存数据。 这个localCache的类型在BaseExecutor内部是写死的。 这个localCache就是一级缓存！**


接下来我们看下**为何执行新增或更新或删除操作，一级缓存就会被清除**这个问题。

首先MyBatis处理新增或删除的时候，最终都是调用update方法，也就是说**新增或者删除操作在MyBatis眼里都是一个更新操作。**

我们看下DefaultSqlSession的update方法：

	public int update(String statement, Object parameter) {
	    try {
	      dirty = true;
	      MappedStatement ms = configuration.getMappedStatement(statement);
	      return executor.update(ms, wrapCollection(parameter));
	    } catch (Exception e) {
	      throw ExceptionFactory.wrapException("Error updating database.  Cause: " + e, e);
	    } finally {
	      ErrorContext.instance().reset();
	    }
    }

很明显，这里调用了CachingExecutor的update方法：

	public int update(MappedStatement ms, Object parameterObject) throws SQLException {
	    flushCacheIfRequired(ms);
	    return delegate.update(ms, parameterObject);
    }

这里的flushCacheIfRequired方法清除的是二级缓存，我们之后会分析。 CachingExecutor委托给了(之前已经分析过)SimpleExecutor的update方法，SimpleExecutor没有Override父类BaseExecutor的update方法，因此我们看BaseExecutor的update方法：

	public int update(MappedStatement ms, Object parameter) throws SQLException {
	    ErrorContext.instance().resource(ms.getResource()).activity("executing an update").object(ms.getId());
	    if (closed) throw new ExecutorException("Executor was closed.");
	    clearLocalCache();
	    return doUpdate(ms, parameter);
    }

我们看到了关键的一句代码： clearLocalCache();  进去看看：
	
	public void clearLocalCache() {
	    if (!closed) {
	      localCache.clear();
	      localOutputParameterCache.clear();
	    }
    }

没错，就是这条，**sqlsession没有关闭的话，进行新增、删除、修改操作的话就是清除一级缓存，也就是SqlSession的缓存。**


## 二级缓存 ##

二级缓存的作用域是全局，换句话说，二级缓存已经脱离SqlSession的控制了。

在测试二级缓存之前，我先把结论说一下：

**二级缓存的作用域是全局的，二级缓存在SqlSession关闭或提交之后才会生效。**

在分析MyBatis的二级缓存之前，我们先简单看下MyBatis中一个关于二级缓存的类(其他相关的类和接口之前已经分析过)：

org.apache.ibatis.mapping.MappedStatement：

MappedStatement类在Mybatis框架中用于表示XML文件中一个sql语句节点，即一个&lt;select /&gt;、&lt;update /&gt;或者&lt;insert /&gt;标签。Mybatis框架在初始化阶段会对XML配置文件进行读取，将其中的sql语句节点对象化为一个个MappedStatement对象。


### 配置 ###

二级缓存跟一级缓存不同，一级缓存不需要配置任何东西，且默认打开。 二级缓存就需要配置一些东西。

本文就说下最简单的配置，在mapper文件上加上这句配置即可：
	
	<cache/>

其实二级缓存跟3个配置有关：

1. mybatis全局配置文件中的setting中的cacheEnabled需要为true(默认为true，不设置也行)
2. mapper配置文件中需要加入&lt;cache&gt;节点
3. mapper配置文件中的select节点需要加上属性useCache需要为true(默认为true，不设置也行)


### 测试 ###

不同SqlSession，查询相同语句，第一次查询之后commit SqlSession：

	@Test
    public void testCache2() {
        SqlSession sqlSession = sqlSessionFactory.openSession();
        SqlSession sqlSession2 = sqlSessionFactory.openSession();
        try {
            String sql = "org.format.mybatis.cache.UserMapper.getById";
            User user = (User)sqlSession.selectOne(sql, 1);
            log.debug(user);
			// 注意，这里一定要提交。 不提交还是会查询两次数据库
            sqlSession.commit();
            User user2 = (User)sqlSession2.selectOne(sql, 1);
            log.debug(user2);
        } finally {
            sqlSession.close();
            sqlSession2.close();
        }
    }

MyBatis仅进行了一次数据库查询：

	==>  Preparing: select * from USERS WHERE ID = ? 
    ==> Parameters: 1(Integer)
    <==      Total: 1
    User{id=1, name='format', age=23, birthday=Sun Oct 12 23:20:13 CST 2014}
    User{id=1, name='format', age=23, birthday=Sun Oct 12 23:20:13 CST 2014}
    
    
不同SqlSession，查询相同语句，第一次查询之后close SqlSession：

	@Test
    public void testCache2() {
        SqlSession sqlSession = sqlSessionFactory.openSession();
        SqlSession sqlSession2 = sqlSessionFactory.openSession();
        try {
            String sql = "org.format.mybatis.cache.UserMapper.getById";
            User user = (User)sqlSession.selectOne(sql, 1);
            log.debug(user);
            sqlSession.close();
            User user2 = (User)sqlSession2.selectOne(sql, 1);
            log.debug(user2);
        } finally {
            sqlSession2.close();
        }
    }

MyBatis仅进行了一次数据库查询：

	==>  Preparing: select * from USERS WHERE ID = ? 
    ==> Parameters: 1(Integer)
    <==      Total: 1
    User{id=1, name='format', age=23, birthday=Sun Oct 12 23:20:13 CST 2014}
    User{id=1, name='format', age=23, birthday=Sun Oct 12 23:20:13 CST 2014}

	
不同SqlSesson，查询相同语句。 第一次查询之后SqlSession不提交：

	@Test
    public void testCache2() {
        SqlSession sqlSession = sqlSessionFactory.openSession();
        SqlSession sqlSession2 = sqlSessionFactory.openSession();
        try {
            String sql = "org.format.mybatis.cache.UserMapper.getById";
            User user = (User)sqlSession.selectOne(sql, 1);
            log.debug(user);
            User user2 = (User)sqlSession2.selectOne(sql, 1);
            log.debug(user2);
        } finally {
            sqlSession.close();
            sqlSession2.close();
        }
    }
    
MyBatis执行了两次数据库查询：

	==>  Preparing: select * from USERS WHERE ID = ? 
    ==> Parameters: 1(Integer)
    <==      Total: 1
    User{id=1, name='format', age=23, birthday=Sun Oct 12 23:20:13 CST 2014}
    ==>  Preparing: select * from USERS WHERE ID = ? 
    ==> Parameters: 1(Integer)
    <==      Total: 1
    User{id=1, name='format', age=23, birthday=Sun Oct 12 23:20:13 CST 2014}

### 源码分析 ###

我们从在mapper文件中加入的&lt;cache/&gt;中开始分析源码，关于MyBatis的SQL解析请参考另外一篇博客[Mybatis解析动态sql原理分析](http://www.cnblogs.com/fangjian0423/p/mybaits-dynamic-sql-analysis.html)。接下来我们看下这个cache的解析：

XMLMappedBuilder（解析每个mapper配置文件的解析类，每一个mapper配置都会实例化一个XMLMapperBuilder类）的解析方法：

	private void configurationElement(XNode context) {
        try {
          String namespace = context.getStringAttribute("namespace");
          if (namespace.equals("")) {
              throw new BuilderException("Mapper's namespace cannot be empty");
          }
          builderAssistant.setCurrentNamespace(namespace);
          cacheRefElement(context.evalNode("cache-ref"));
          cacheElement(context.evalNode("cache"));
          parameterMapElement(context.evalNodes("/mapper/parameterMap"));
          resultMapElements(context.evalNodes("/mapper/resultMap"));
          sqlElement(context.evalNodes("/mapper/sql"));
          buildStatementFromContext(context.evalNodes("select|insert|update|delete"));
        } catch (Exception e) {
          throw new BuilderException("Error parsing Mapper XML. Cause: " + e, e);
        }
	}
    
我们看到了解析cache的那段代码：

	private void cacheElement(XNode context) throws Exception {
        if (context != null) {
          String type = context.getStringAttribute("type", "PERPETUAL");
          Class<? extends Cache> typeClass = typeAliasRegistry.resolveAlias(type);
          String eviction = context.getStringAttribute("eviction", "LRU");
          Class<? extends Cache> evictionClass = typeAliasRegistry.resolveAlias(eviction);
          Long flushInterval = context.getLongAttribute("flushInterval");
          Integer size = context.getIntAttribute("size");
          boolean readWrite = !context.getBooleanAttribute("readOnly", false);
          Properties props = context.getChildrenAsProperties();
          builderAssistant.useNewCache(typeClass, evictionClass, flushInterval, size, readWrite, props);
        }
	}
    
解析完cache标签之后会使用builderAssistant的userNewCache方法，这里的builderAssistant是一个MapperBuilderAssistant类型的帮助类，每个XMLMappedBuilder构造的时候都会实例化这个属性，MapperBuilderAssistant类内部有个Cache类型的currentCache属性，这个属性也就是mapper配置文件中cache节点所代表的值：

	public Cache useNewCache(Class<? extends Cache> typeClass,
      Class<? extends Cache> evictionClass,
      Long flushInterval,
      Integer size,
      boolean readWrite,
      Properties props) {
        typeClass = valueOrDefault(typeClass, PerpetualCache.class);
        evictionClass = valueOrDefault(evictionClass, LruCache.class);
        Cache cache = new CacheBuilder(currentNamespace)
            .implementation(typeClass)
            .addDecorator(evictionClass)
            .clearInterval(flushInterval)
            .size(size)
            .readWrite(readWrite)
            .properties(props)
            .build();
        configuration.addCache(cache);
        currentCache = cache;
        return cache;
	}
    
ok，现在mapper配置文件中的cache节点被解析到了XMLMapperBuilder实例中的builderAssistant属性中的currentCache值里。

接下来XMLMapperBuilder会解析select节点，解析select节点的时候使用XMLStatementBuilder进行解析(也包括其他insert，update，delete节点)：

	public void parseStatementNode() {
	    String id = context.getStringAttribute("id");
	    String databaseId = context.getStringAttribute("databaseId");
	
	    if (!databaseIdMatchesCurrent(id, databaseId, this.requiredDatabaseId)) return;
	
	    Integer fetchSize = context.getIntAttribute("fetchSize");
	    Integer timeout = context.getIntAttribute("timeout");
	    String parameterMap = context.getStringAttribute("parameterMap");
	    String parameterType = context.getStringAttribute("parameterType");
	    Class<?> parameterTypeClass = resolveClass(parameterType);
	    String resultMap = context.getStringAttribute("resultMap");
	    String resultType = context.getStringAttribute("resultType");
	    String lang = context.getStringAttribute("lang");
	    LanguageDriver langDriver = getLanguageDriver(lang);
	
	    Class<?> resultTypeClass = resolveClass(resultType);
	    String resultSetType = context.getStringAttribute("resultSetType");
	    StatementType statementType = StatementType.valueOf(context.getStringAttribute("statementType", StatementType.PREPARED.toString()));
	    ResultSetType resultSetTypeEnum = resolveResultSetType(resultSetType);
	
	    String nodeName = context.getNode().getNodeName();
	    SqlCommandType sqlCommandType = SqlCommandType.valueOf(nodeName.toUpperCase(Locale.ENGLISH));
	    boolean isSelect = sqlCommandType == SqlCommandType.SELECT;
	    boolean flushCache = context.getBooleanAttribute("flushCache", !isSelect);
	    boolean useCache = context.getBooleanAttribute("useCache", isSelect);
	    boolean resultOrdered = context.getBooleanAttribute("resultOrdered", false);
	
	    // Include Fragments before parsing
	    XMLIncludeTransformer includeParser = new XMLIncludeTransformer(configuration, builderAssistant);
	    includeParser.applyIncludes(context.getNode());
	
	    // Parse selectKey after includes and remove them.
	    processSelectKeyNodes(id, parameterTypeClass, langDriver);
	    
	    // Parse the SQL (pre: <selectKey> and <include> were parsed and removed)
	    SqlSource sqlSource = langDriver.createSqlSource(configuration, context, parameterTypeClass);
	    String resultSets = context.getStringAttribute("resultSets");
	    String keyProperty = context.getStringAttribute("keyProperty");
	    String keyColumn = context.getStringAttribute("keyColumn");
	    KeyGenerator keyGenerator;
	    String keyStatementId = id + SelectKeyGenerator.SELECT_KEY_SUFFIX;
	    keyStatementId = builderAssistant.applyCurrentNamespace(keyStatementId, true);
	    if (configuration.hasKeyGenerator(keyStatementId)) {
	      keyGenerator = configuration.getKeyGenerator(keyStatementId);
	    } else {
	      keyGenerator = context.getBooleanAttribute("useGeneratedKeys",
	          configuration.isUseGeneratedKeys() && SqlCommandType.INSERT.equals(sqlCommandType))
	          ? new Jdbc3KeyGenerator() : new NoKeyGenerator();
	    }
	
	    builderAssistant.addMappedStatement(id, sqlSource, statementType, sqlCommandType,
	        fetchSize, timeout, parameterMap, parameterTypeClass, resultMap, resultTypeClass,
	        resultSetTypeEnum, flushCache, useCache, resultOrdered, 
	        keyGenerator, keyProperty, keyColumn, databaseId, langDriver, resultSets);
	}

这段代码前面都是解析一些标签的属性，我们看到了最后一行使用builderAssistant添加MappedStatement，其中builderAssistant属性是构造XMLStatementBuilder的时候通过XMLMappedBuilder传入的，我们继续看builderAssistant的addMappedStatement方法：

![](http://images.cnblogs.com/cnblogs_com/fangjian0423/603237/o_mybatis_cache02.jpg)	
	
进入setStatementCache：

	private void setStatementCache(
      boolean isSelect,
      boolean flushCache,
      boolean useCache,
      Cache cache,
      MappedStatement.Builder statementBuilder) {
	    flushCache = valueOrDefault(flushCache, !isSelect);
	    useCache = valueOrDefault(useCache, isSelect);
	    statementBuilder.flushCacheRequired(flushCache);
	    statementBuilder.useCache(useCache);
	    statementBuilder.cache(cache);
	}

最终mapper配置文件中的&lt;cache/&gt;被设置到了XMLMapperBuilder的builderAssistant属性中，XMLMapperBuilder中使用XMLStatementBuilder遍历CRUD节点，遍历CRUD节点的时候将这个cache节点设置到这些CRUD节点中，这个cache就是所谓的二级缓存！

接下来我们回过头来看查询的源码，CachingExecutor的query方法：
    
![](http://images.cnblogs.com/cnblogs_com/fangjian0423/603237/o_mybatis_cache03.jpg)	

进入TransactionalCacheManager的putObject方法：

	public void putObject(Cache cache, CacheKey key, Object value) {
    	getTransactionalCache(cache).putObject(key, value);
	}


	private TransactionalCache getTransactionalCache(Cache cache) {
	    TransactionalCache txCache = transactionalCaches.get(cache);
	    if (txCache == null) {
	      txCache = new TransactionalCache(cache);
	      transactionalCaches.put(cache, txCache);
	    }
	    return txCache;
    }

TransactionalCache的putObject方法：

	public void putObject(Object key, Object object) {
	    entriesToRemoveOnCommit.remove(key);
	    entriesToAddOnCommit.put(key, new AddEntry(delegate, key, object));
    }

我们看到，数据被加入到了entriesToAddOnCommit中，这个entriesToAddOnCommit是什么东西呢，它是TransactionalCache的一个Map属性：

	private Map<Object, AddEntry> entriesToAddOnCommit;

AddEntry是TransactionalCache内部的一个类：

	private static class AddEntry {
	    private Cache cache;
	    private Object key;
	    private Object value;
	
	    public AddEntry(Cache cache, Object key, Object value) {
	      this.cache = cache;
	      this.key = key;
	      this.value = value;
	    }
	
	    public void commit() {
	      cache.putObject(key, value);
	    }
	}


好了，现在我们发现使用二级缓存之后：查询数据的话，先从二级缓存中拿数据，如果没有的话，去一级缓存中拿，一级缓存也没有的话再查询数据库。有了数据之后在丢到TransactionalCache这个对象的entriesToAddOnCommit属性中。

**接下来我们来验证为什么SqlSession commit或close之后，二级缓存才会生效这个问题。**

DefaultSqlSession的commit方法：

	public void commit(boolean force) {
	    try {
	      executor.commit(isCommitOrRollbackRequired(force));
	      dirty = false;
	    } catch (Exception e) {
	      throw ExceptionFactory.wrapException("Error committing transaction.  Cause: " + e, e);
	    } finally {
	      ErrorContext.instance().reset();
	    }
    }

CachingExecutor的commit方法：

	public void commit(boolean required) throws SQLException {
	    delegate.commit(required);
	    tcm.commit();
	    dirty = false;
    }

tcm.commit即 TransactionalCacheManager的commit方法：

	public void commit() {
	    for (TransactionalCache txCache : transactionalCaches.values()) {
	      txCache.commit();
	    }
    }

TransactionalCache的commit方法：

	public void commit() {
	    delegate.getReadWriteLock().writeLock().lock();
	    try {
	      if (clearOnCommit) {
	        delegate.clear();
	      } else {
	        for (RemoveEntry entry : entriesToRemoveOnCommit.values()) {
	          entry.commit();
	        }
	      }
	      for (AddEntry entry : entriesToAddOnCommit.values()) {
	        entry.commit();
	      }
	      reset();
	    } finally {
	      delegate.getReadWriteLock().writeLock().unlock();
	    }
    }

发现调用了AddEntry的commit方法：

	public void commit() {
      cache.putObject(key, value);
    }

发现了！ AddEntry的commit方法会把数据丢到cache中，也就是丢到二级缓存中！

关于为何调用close方法后，二级缓存才会生效，因为close方法内部会调用commit方法。本文就不具体说了。 读者有兴趣的话看一看源码就知道为什么了。

## 其他 ##

### Cache接口简介 ###

org.apache.ibatis.cache.Cache是MyBatis的缓存接口，想要实现自定义的缓存需要实现这个接口。

MyBatis中关于Cache接口的实现类也使用了装饰者设计模式。

我们看下它的一些实现类：

![](http://images.cnblogs.com/cnblogs_com/fangjian0423/603237/o_mybatis_cache04.jpg)	

简单说明：

LRU – 最近最少使用的:移除最长时间不被使用的对象。

FIFO – 先进先出:按对象进入缓存的顺序来移除它们。

SOFT – 软引用:移除基于垃圾回收器状态和软引用规则的对象。

WEAK – 弱引用:更积极地移除基于垃圾收集器状态和弱引用规则的对象。

	<cache
	  eviction="FIFO"
	  flushInterval="60000"
	  size="512"
	  readOnly="true"/>

可以通过cache节点的eviction属性设置，也可以设置其他的属性。

### cache-ref节点 ###

mapper配置文件中还可以加入cache-ref节点，它有个属性namespace。

如果每个mapper文件都是用cache-ref，且namespace都一样，那么就代表着真正意义上的全局缓存。

如果只用了cache节点，那仅代表这个这个mapper内部的查询被缓存了，其他mapper文件的不起作用，这并不是所谓的全局缓存。


## 总结 ##

总体来说，MyBatis的源码看起来还是比较轻松的，本文从实践和源码方面深入分析了MyBatis的缓存原理，希望对读者有帮助。
