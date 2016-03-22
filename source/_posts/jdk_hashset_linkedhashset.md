title: jdk源码分析之HashSet, LinkedHashSet
date: 2015-02-21 23:43:37
tags:
- jdk
- set
categories: jdk
description: 合跟数组不一样，数组中的元素可以重复，而集合中的元素却不可以重复 ...
----------------

## 前言 ##

Set是一个集合。

集合跟数组不一样，数组中的元素可以重复，而集合中的元素却不可以重复。

## HashSet的源码分析 ##

首先看下HashSet的属性。

HashSet只有两个属性：

    private transient HashMap<E,Object> map;

    private static final Object PRESENT = new Object();

1个map对象，这个map的key为泛型类型，也就是声明Set的那个泛型，value是一个Object类型。

PRESENT是一个静态final类型的Object实例，final类型也表示这个实例不会再次被实例化。

接下来看下HashSet的几个重要方法。


### boolean add(E e) ###

** 添加元素到HashSet集合里 **

    public boolean add(E e) {
        return map.put(e, PRESENT)==null;
    }

可以看到，添加元素到集合里的方法是将元素作为key，然后以PRESENT为value，丢入到map里的。

这样的话，当添加重复的元素的时候，由于HashMap中对于有重复的键值，不会新增一个项，因此HashSet中的重复元素不会被添加。


### boolean remove(Object o) ###

** 删除集合中的元素 **

	public boolean remove(Object o) {
        return map.remove(o)==PRESENT;
    }

调用map的remove方法。


### boolean contains(Object o) ###

** 查看集合中是否存在对象的元素 **

	public boolean contains(Object o) {
        return map.containsKey(o);
    }
    
使用hashmap的containsKey方法。

## HashSet总结 ##

1. HashSet内部使用HashMap，HashSet集合内部所有的操作基本上都是基于HashMap完成的
2. HashSet中的元素是无序的，这是因为它内部使用HashMap进行存储，而HashMap添加键值对的时候是根据hash函数得到数组的下标的

## LinkedHashSet源码分析 ##

LinkedHashSet继承自HashSet，仅仅覆盖了spliterator方法。

	public LinkedHashSet(int initialCapacity, float loadFactor) {
        super(initialCapacity, loadFactor, true);
    }

    public LinkedHashSet(int initialCapacity) {
        super(initialCapacity, .75f, true);
    }

    public LinkedHashSet() {
        super(16, .75f, true);
    }

    public LinkedHashSet(Collection<? extends E> c) {
        super(Math.max(2*c.size(), 11), .75f, true);
        addAll(c);
    }

LinkedHashSet的4个构造方法都调用了父类的同一个方法，也就是HashSet的构造方法：

	HashSet(int initialCapacity, float loadFactor, boolean dummy) {
        map = new LinkedHashMap<>(initialCapacity, loadFactor);
    }
    
HashSet的这个构造方法将内部的HashMap属性用LinkedHashMap构造。 由于LinkedHashMap的这个构造方法的迭代顺序是插入顺序，因此LinkedHashSet也就是一个会记录插入顺序的集合。

## LinkedHashSet总结 ##

1. LinkedHashSet继自HashSet，仅仅覆盖了spliterator方法，其他方法都跟HashSet相同
2. LinkedHashSet的4个人构造方法都调用了同一个HashSet的构造方法，HashSet的这个构造方法会把内部的HashMap属性实例化成LinkedHashMap。 所以说LinkedHashSet内部是使用LinkedHashMap完成各个操作的。

