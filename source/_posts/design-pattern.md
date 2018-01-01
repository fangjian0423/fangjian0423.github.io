title: 记录自己理解的一些设计模式
date: 2017-03-26 15:33:29
tags:
- architecture
- java
categories: architecture

----------------

记录一下自己理解的一些设计模式，并尽量使用表达清楚的例子进行讲解。

<!--more-->

## 策略模式

策略模式应该是最基础的一个设计模式，它是对行为的一个抽象。jdk中的Comparator比较器就是一个使用策略设计模式的策略。

比如有一个Student学生类，有name和age两个属性。如果有个需求需要打印学生名单，并按照字母顺序排序，可以使用Comparator接口并在内部使用name进行比较即可。 如果哪一天需要按照年龄进行排序，那么只需要修改Comparator即可，也就是使用一个新的策略，其它完全不变。

## 工厂模式

工厂模式的意义在于对象的创建、管理可以使用工厂去管理，而不是创建者自身。最典型的工厂模式使用者就是Spring，Spring内部的容器就是一个工厂，所有的bean都由这个容器管理，包括它们的创建、销毁、注入都被这个容器管理。

工厂模式分简单工厂和抽象工厂。它们的区别在于抽象工厂抽象程度更高，把工厂也抽象成了一个接口，这样可以再每添加一个新的对象的时候而不需要修改工厂的代码。

比如有个Repository接口，用于存储数据，有DatabaseRepository，CacheRepository，FileRepository分别在数据库，缓存，文件中存储数据，定义如下：

    public interface Repository {
        void save(Object obj);
    }

    class DatabaseRepository implements Repository {
        @Override
        public void save(Object obj) {
            System.out.println("save in database");
        }
    }
    class CacheRepository implements Repository {
        @Override
        public void save(Object obj) {
            System.out.println("save in cache");
        }
    }
    class FileRepository implements Repository {
        @Override
        public void save(Object obj) {
            System.out.println("save in file");
        }
    }

### 简单工厂的使用

    public class RepositoryFactory {

        public Repository create(String type) {
            Repository repository = null;
            switch (type) {
                case "db":
                    repository = new DatabaseRepository();
                    break;
                case "cache":
                    repository = new CacheRepository();
                    break;
                case "file":
                    repository = new FileRepository();
                    break;
            }
            return repository;
        }

        public static void main(String[] args) {
            RepositoryFactory factory = new RepositoryFactory();
            factory.create("db").save(new Object());
            factory.create("cache").save(new Object());
            factory.create("file").save(new Object());
        }
    }

简单工厂的弊端在于每添加一个新的Repository，都必须修改RepositoryFactory中的代码

### 抽象工厂的使用

    public interface RepositoryFactoryProvider {
        Repository create();
    }

    class DatabaseRepositoryFactory implements RepositoryFactoryProvider {
        @Override
        public Repository create() {
            return new DatabaseRepository();
        }
    }
    class CacheRepositoryFactory implements RepositoryFactoryProvider {
        @Override
        public Repository create() {
            return new CacheRepository();
        }
    }
    class FileRepositoryFactory implements RepositoryFactoryProvider {
        @Override
        public Repository create() {
            return new FileRepository();
        }
    }

抽象工厂的测试：

    RepositoryFactoryProvider dbProvider = new DatabaseRepositoryFactory();
    dbProvider.create().save(new Object());
    RepositoryFactoryProvider cacheProvider = new CacheRepositoryFactory();
    cacheProvider.create().save(new Object());
    RepositoryFactoryProvider fileProvider = new FileRepositoryFactory();
    fileProvider.create().save(new Object());

抽象工厂把工厂也进行了抽象话，所以添加一个新的Repository的话，只需要新增一个RepositoryFactory即可，原有代码不需要修改。

## 装饰者模式

装饰者模式的作用就在于它可以在不改变原有类的基础上动态地给类添加新的功能。之前写过一篇[通过源码分析MyBatis的缓存](http://www.cnblogs.com/fangjian0423/p/mybatis-cache.html)文章，mybatis中的query就是使用了装饰者设计模式。

用一段简单的代码来模拟一下mybatis中query的实现原理：

    @Data
    @AllArgsConstructor
    @ToString
    class Result { // 查询结果类，相当于一个domain
      private Object obj;
      private String sql;
    }

    public interface Query { // 查询接口，有简单查询和缓存查询
      Result query(String sql);
    }

    public class SimpleQuery implements Query { // 简单查询，相当于直接查询数据库，这里直接返回Result，相当于是数据库查询的结果
      @Override
      public Result query(String sql) {
          return new Result(new Object(), sql);
      }
    }

    public class CacheQuery implements Query { // 缓存查询，如果查询相同的sql，不直接查询数据库，而是返回map中存在的Result
      private Query query;
      private Map<String, Result> cache = new HashMap<>();
      public CacheQuery(Query query) {
          this.query = query;
      }
      @Override
      public Result query(String sql) {
          if(cache.containsKey(sql)) {
              return cache.get(sql);
          }
          Result result = query.query(sql);
          cache.put(sql, result);
          return result;
      }
    }

测试：

    Query simpleQuery = new SimpleQuery();
    System.out.println(simpleQuery.query("select * from t_student") == simpleQuery.query("select * from t_student")); // false
    Query cacheQuery = new CacheQuery(simpleQuery);
    System.out.println(cacheQuery.query("select * from t_student") == cacheQuery.query("select * from t_student")); // true

这里CacheQuery就是一个装饰类，SimpleQuery是一个被装饰者。我们通过装饰者设计模式动态地给SimpleQuery添加了缓存功能，而不需要修改SimpleQuery的代码。

当然，装饰者模式也有缺点，就是会存在太多的类。

如果我们需要添加一个过滤的查询(sql中有敏感字的就直接返回null，而不查询数据库)，只需要可以添加一个FilterQuery装饰者即可：

    public class FilterQuery implements Query {
        private Query query;
        private List<String> words = new ArrayList<>();
        public FilterQuery(Query query) {
            this.query = query;
            words.add("fuck");
            words.add("sex");
        }
        @Override
        public Result query(String sql) {
            for(String word : words) {
                if(sql.contains(word)) return null;
            }
            return query.query(sql);
        }
    }

    Query filterQuery = new FilterQuery(simpleQuery);
    System.out.println(filterQuery.query("select * from t_student where name = 'fuck'"));  // null
    System.out.println(filterQuery.query("select * from t_student where name = 'format'")); // Result(obj=java.lang.Object@1b4fb997, sql=select * from t_student where name = 'format')

## 代理模式

代理模式的作用是使用一个代理类来代替原先类进行操作。比较常见的就是aop中就是使用代理模式完成事务的处理。

代理模式分静态代理和动态代理，静态代理的原理就是对目标对象进行封装，最后调用目标对象的方法即可。

动态代理跟静态代理的区别就是动态代理中的代理类是程序运行的时候生成的。Spring中对于接口的代理使用jdk内置的Proxy和InvocationHandler实现，对于类的代理使用cglib完成。

以1个UserService为例，使用jdk自带的代理模式完成计算方法调用时间的需求：

    // UserService接口
    public interface IUserService {
        void printAll();
    }
    // UserService实现类
    class UserService implements IUserService {
        @Override
        public void printAll() {
            System.out.println("print all users");
        }
    }
    // InvocationHandler策略，这里打印了方法调用前后的时间
    @AllArgsConstructor
    class UserInvocationHandler implements InvocationHandler {
        private IUserService userService;
        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            System.out.println("start : " + System.currentTimeMillis());
            Object result = method.invoke(userService, args);
            System.out.println("end : " + System.currentTimeMillis());
            return result;
        }
    }

测试：

    IUserService userService = new UserService();
    UserInvocationHandler uih = new UserInvocationHandler(userService);
    IUserService proxy = (IUserService) Proxy.newProxyInstance(userService.getClass().getClassLoader(), new Class[] {IUserService.class}, uih);
    proxy.printAll(); // 打印出start : 1489665566456  print all users  end : 1489665566457

## 组合模式

组合模式经常跟策略模式配合使用，用来组合所有的策略，并遍历这些策略找出满足条件的策略。之前写过一篇[SpringMVC关于json、xml自动转换的原理研究](http://www.cnblogs.com/fangjian0423/p/springMVC-xml-json-convert.html)文章，里面springmvc把返回的返回值映射给用户的response做了一层抽象，封装到了HandlerMethodReturnValueHandler策略接口中。

在HandlerMethodReturnValueHandlerComposite类中，使用存在的HandlerMethodReturnValueHandler对返回值进行处理，在HandlerMethodReturnValueHandlerComposite内部的代码如下：

    // 策略集合
    private final List<HandlerMethodReturnValueHandler> returnValueHandlers = new ArrayList<HandlerMethodReturnValueHandler>();

    @Override
    public void handleReturnValue(Object returnValue, MethodParameter returnType,
        ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {
        // 调用selectHandler方法
        HandlerMethodReturnValueHandler handler = selectHandler(returnValue, returnType);
        if (handler == null) {
          throw new IllegalArgumentException("Unknown return value type: " + returnType.getParameterType().getName());
        }
        handler.handleReturnValue(returnValue, returnType, mavContainer, webRequest); // 使用找到的handler进行处理
    }

    private HandlerMethodReturnValueHandler selectHandler(Object value, MethodParameter returnType) {
        boolean isAsyncValue = isAsyncReturnValue(value, returnType);
        // 遍历存在的HandlerMethodReturnValueHandler
        for (HandlerMethodReturnValueHandler handler : this.returnValueHandlers) {
          if (isAsyncValue && !(handler instanceof AsyncHandlerMethodReturnValueHandler)) {
            continue;
          }
          if (handler.supportsReturnType(returnType)) { // 找到匹配的handler
            return handler;
          }
        }
        return null;
    }


## 模板模式

跟策略模式类似，模板模式会先定义好实现的逻辑步骤，但是具体的实现方式由子类完成，跟策略模式的区别就是模板模式是有逻辑步骤的。比如要给院系里的学生排序，并取出排名第一的学生。这里就有2个步骤，分别是排序和取出第一名学生。

一段伪代码：

    public abstract class AbstractStudentGetter {
        public final Student getStudent(List<Student> students) {
            sort(students); // 第一步
            if(!CollectionUtils.isEmpty(students)) {
                return students.get(0);  // 第二步
            }
            return null;
        }
        abstract public void sort(List<Student> students);
    }
    class AgeStudentGetter extends AbstractStudentGetter { // 取出年纪最大的学生
        @Override
        public void sort(List<Student> students) {
            students.sort(new Comparator<Student>() {
                @Override
                public int compare(Student s1, Student s2) {
                    return s2.getAge() - s1.getAge();
                }
            });
        }
    }
    class NameStudentGetter extends AbstractStudentGetter { // 按照名字字母排序取出第一个学生
        @Override
        public void sort(List<Student> students) {
            students.sort(new Comparator<Student>() {
                @Override
                public int compare(Student s1, Student s2) {
                    return s2.getName().compareTo(s1.getName());
                }
            });
        }
    }

测试：

    AbstractStudentGetter ageGetter = new AgeStudentGetter();
    AbstractStudentGetter nameGetter = new NameStudentGetter();

    List<Student> students = new ArrayList<>();
    students.add(new Student("jim", 22));
    students.add(new Student("format", 25));

    System.out.println(ageGetter.getStudent(students)); // Student(name=format, age=25)
    System.out.println(nameGetter.getStudent(students)); // Student(name=jim, age=22)


## 观察者设计模式

观察者设计模式主要的使用场景在于一个对象变化之后，依赖该对象的对象会收到通知。典型的例子就是rss的订阅，当订阅了博客的rss之后，当博客更新之后，订阅者就会收到新的订阅信息。

jdk内置提供了Observable和Observer，用来实现观察者模式：

    // 定义一个Observable
    public class MetricsObserable extends Observable {
        private Map<String, Long> counterMap = new HashMap<>();
        public void updateCounter(String key, Long value) {
            counterMap.put(key, value);
            setChanged();
            notifyObservers(counterMap);
        }
    }
    // Observer
    public class AdminA implements Observer {
        @Override
        public void update(Observable o, Object arg) {
            System.out.println("adminA: " + arg);
        }
    }
    public class AdminB implements Observer {
        @Override
        public void update(Observable o, Object arg) {
            System.out.println("adminB: " + arg);
        }
    }

测试：

    MetricsObserable metricsObserable = new MetricsObserable();
    metricsObserable.addObserver(new AdminA());
    metricsObserable.addObserver(new AdminB());
    metricsObserable.updateCounter("request-count", 100l);

打印出：

    adminB: {request-count=100}
    adminA: {request-count=100}

## 享元模式

线程池中会构造几个核心线程用于处理，这些线程会去取阻塞队列里的任务然后进行执行。这些线程就是会被共享、且被重复使用的。因为线程的创建、销毁、调度都是需要消耗资源的，没有必要每次创建新的线程，而是共用一些线程。这就是享元模式的使用。类似的还有jdbc连接池，对象池等。

之前有一次面试被问到：

    Integer.valueOf("1") == Integer.valueOf("1") // true还是false

当时回答的是false，后来翻了下Integer的源码发现Integer里面有个内部类IntegerCache，用于缓存一些共用的Integer。这个缓存的范围可以在jvm启动的时候进行设置。

其实后来想想也应该这么做，我们没有必要每次使用对象的时候都返回新的对象，可以共享这些对象，因为新对象的创建都是需要消耗内存的。


## 适配器模式

适配器模式比较好理解。像生活中插线口的插头有2个口的，也有3个口的。如果电脑的电源插口只有3个口的，但是我们需要一个2个口的插口的话，这个时候就需要使用插座来外接这个3个口的插头，插座上有2个口的插头。

这个例子跟我们编程一样，当用户系统的接口跟我们系统内部的接口不一致时，我们可以使用适配器来完成接口的转换。

使用继承的方式实现类的适配：

    public class Source {
        public void method() {
            System.out.println("source method");
        }
    }
    interface Targetable {
        void method();
        void newMethod();
    }
    class Adapter extends Source implements Targetable {
        @Override
        public void newMethod() {
            System.out.println("new method");
        }
    }

测试：

    Targetable targetable = new Adapter();
    targetable.method(); // source method
    targetable.newMethod(); // new method

上述方式是用接口和继承的方式实现适配器模式。当然我们也可以使用组合的方式实现(把Source当成属性放到Adapter中)。

## 单例模式

单例模式比较好理解，Spring就是典型的例子。被Spring中的容器管理的对象都有对应的scope，配置成singleton说明这个对象就是单例，也就是在Spring容器的生命周期中，这个类只有1个实例。

java中单例模式的写法也有好多种。比如懒汉式、饿汉式、内部类方式、枚举方式等。

需要注意的如果使用dcl的话需要初始化过程，这篇[Java内存模型之从JMM角度分析DCL](http://cmsblogs.com/?p=2161&from=timeline&isappinstalled=0)文章中说明了dcl的正确用法。

Effectice java中推荐的单例方式写法是使用枚举类型的方式。

## 外观模式

外观模式用来包装一组接口用于方便使用。 比如系统中分10个模块，有个功能需要组合使用所有的模块，这个时候就需要一个包装类包装这10个接口，然后进行业务逻辑的调用。
