title: google guava类库介绍
date: 2015-07-26 16:32:33
tags:
- guava
- java
categories:
- java
description: Guava是一个Google开发的基于java的扩展项目，提供了很多有用的工具类，可以让java代码更加优雅，更加简洁 ...

---------------

## Guava简介 ##

[Guava](https://mail.google.com/mail/u/0/)是一个Google开发的基于java的扩展项目，提供了很多有用的工具类，可以让java代码更加优雅，更加简洁。

Guava包括诸多工具类，比如Collections，cache，concurrent，hash，reflect，annotations，eventbus等。

刚好在看flume源码的时候看到源码里面使用了很多guava提供的代码，于是记录学习一下这个类库。

## 各个模块介绍 ##

[Guava的wiki](https://code.google.com/p/guava-libraries/wiki/GuavaExplained)已经很明细地介绍了各个工具类的作用和说明。

简单翻译一下各个工具类的说明，有用到的需要了解详情的直接去官网看就可以了。

### Basic utilties 基础工具类 ###

基础工具类的作用是写java语言写的更轻松。它包括了5个子模块：

1.[使用和避免null](https://code.google.com/p/guava-libraries/wiki/UsingAndAvoidingNullExplained)，null值是有歧义的，也会引起错误。有时候它会让人很不舒服，
2.[前置条件](https://code.google.com/p/guava-libraries/wiki/PreconditionsExplained),让方法中的条件检查更简单
3.[公用的object方法](https://code.google.com/p/guava-libraries/wiki/CommonObjectUtilitiesExplained)，简化object对象的hashCode和toString
4.[排序](https://code.google.com/p/guava-libraries/wiki/OrderingExplained)，Guava提供了强大的fluent Comparator
5.[Throwables](https://code.google.com/p/guava-libraries/wiki/ThrowablesExplained)，简化了异常和错误的传播与检查

### Collections 集合 ###

Guava扩展了jdk提供的集合机制

1.[不可变集合](https://code.google.com/p/guava-libraries/wiki/ImmutableCollectionsExplained)用不变的集合进行防御性编程和性能提升
2.[新集合类型](https://code.google.com/p/guava-libraries/wiki/NewCollectionTypesExplained)multisets，multimaps，tables，bidirectional map等
3.[强大的集合工具类](https://code.google.com/p/guava-libraries/wiki/CollectionUtilitiesExplained)提供了jdk中没有的集合工具类
4.[扩展工具类](https://code.google.com/p/guava-libraries/wiki/CollectionHelpersExplained)让实现和扩展集合类变得更容易，比如创建Collection的装饰器，或实现迭代器


### Caches 缓存 ###

本地缓存实现，支持多种缓存过期策略

### 函数式风格 ###

Guava的函数式支持可以显著简化代码，但请谨慎使用它

### 并发 ###

1.[ListenableFuture](https://code.google.com/p/guava-libraries/wiki/ListenableFutureExplained)：完成后触发回调的Future
2.[Service框架](https://code.google.com/p/guava-libraries/wiki/ServiceExplained)：抽象可开启和关闭的服务，帮助你维护服务的状态逻辑


### 字符串处理 ###

非常有用的字符串工具，包括分割、连接、填充等操作

### 原生类型 ###

扩展 JDK 未提供的原生类型（如int、char）操作， 包括某些类型的无符号形式

### 区间 ### 

可比较类型的区间API，包括连续和离散类型

### IO ###

简化I/O尤其是I/O流和文件的操作，针对Java5和6版本

### 散列 ###

提供比Object.hashCode()更复杂的散列实现，并提供布鲁姆过滤器的实现

### 事件总线 ###

发布-订阅模式的组件通信，但组件不需要显式地注册到其他组件中

### 数学运算 ###

优化的、充分测试的数学工具类

### 反射 ###

Guava的Java反射机制工具类

