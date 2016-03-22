title: mybatis-helper介绍
date: 2015-01-06 00:32:24
tags:
- mybatis
categories:
- mybatis
description: 写了个mybatis-helper小东西 ...
----------------

最近没事写了一个[mybatis-helper](https://github.com/fangjian0423/mybatis-helper)这么个小东西。

主要的功能就是扩展一下mybatis，提供了一个BaseDao，这个dao有query，count，getAll，getById，insert，update，delete这些最基础的方法。

有了这个BaseDao，这样的话继承这个dao就会有这些基础方法。只有一个条件，那就是对应实体的xml配置文件中需要有一个对应实体的resultMap，这个resultMap的id必须为"resultMap"。

然后这个helper还提供了一个分页和排序的接口，dto继承默认的接口实现类即可，在查询的时候使用这个dto作为查询条件将会默认带上分页和排序的功能。

这个helper基本的原理：

BaseDao中的所有方法都使用Annotation的方式处理，然后对应的SqlProvider生成sql，其中表名用TABLE表示，其他都根据对应的参数处理。

接下来使用拦截器拦截StatementHandler的时候使用反射得到MappedStatement中的BoundSql，然后根据全局的resultMap处理sql，把TABLE字符串替换成真正的表名。 分页和排序的功能也是在这个拦截器中实现的。

很简单的一个helper，不想做的太复杂，感觉没什么必要。
