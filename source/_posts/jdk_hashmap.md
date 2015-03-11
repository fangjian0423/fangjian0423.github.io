title: jdk源码分析之HashMap
date: 2015-02-13 112:47:32
tags: [jdk,map,HashMap]
description: 介绍jdk内部的HashMap相关的知识
----------------

## 前言 ##

Map是一个映射键和值的对象。类似于Python中的字典。

HashMap为什么会出现呢?

因为数组这种数据结构，虽然遍历简单，但是插入和删除操作复杂，需要移动数组内部的元素；链表这种数据结构，插入和删除操作简单，但是查找复杂，只能一个一个地遍历。

有没有一种新的数据结构，插入数据简单，同时查找也简单？ 这个时候就出现了哈希表这种数据结构。 这是一种折中的方式，插入没数组快，插入没链表快。

哈希表这个东西，学过数据结构的都应该知道。

下图是用拉链法实现的哈希表，Java中的HashMap就是使用这种数据结构。

一个数组，数组上的各个项存储着链表的表头。

![](http://images.cnblogs.com/cnblogs_com/fangjian0423/603237/o_HashMap.png)

## HashMap的源码分析 ##

HashMap是Map接口的重要实现类之一。

HashMap中的key和value都可以为null，且它的方法都没有synchronized。 其他方法的实现大部分跟HashTable一致。HashTable的相关源码不在这里介绍，基本上跟HashTable一致。

HashMap有个内部静态类Node，这个Node就是拉链法哈希表上的数组上存储的链表节点，它有4个属性，hash表示哈希值，key表示键，value表示值, next表示这个节点的下一个节点，它并没有prev节点，这点与之前的截图一致：

	static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
        Node<K,V> next;

        Node(int hash, K key, V value, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }

        public final K getKey()        { return key; }
        public final V getValue()      { return value; }
        public final String toString() { return key + "=" + value; }

        public final int hashCode() {
            return Objects.hashCode(key) ^ Objects.hashCode(value);
        }

        public final V setValue(V newValue) {
            V oldValue = value;
            value = newValue;
            return oldValue;
        }

        public final boolean equals(Object o) {
            if (o == this)
                return true;
            if (o instanceof Map.Entry) {
                Map.Entry<?,?> e = (Map.Entry<?,?>)o;
                if (Objects.equals(key, e.getKey()) &&
                    Objects.equals(value, e.getValue()))
                    return true;
            }
            return false;
        }
    }

这个Node节点类的属性有一个hash值，一个键值key，一个value还有一个表示下一个节点的Node类属性。它实现了Map接口内部的Entry接口。

### HashMap的属性 ###

然后是HashMap的几个重要的属性:

	transient Node<K,V>[] table;

    transient int size;

    transient int modCount;

    int threshold;

    final float loadFactor;

table属性是一个数组，是一个Node类型数组，这就是之前分析的哈希表的结构。size表示这个map中键值对的个数。modCount跟迭代器相关，关于迭代器部分的知识到时候会在一篇文章中介绍。threshold表示阀值，它的值是容量*加载因子。loadFactor表示加载因子。


### HashMap的重要方法 ###

#### V put(K key, V value) ####

** 把一对键值对丢入到HashMap中 **

	public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }

put方法调用putVal方法，并调用了一个hash方法将key转换成了一个hash值。

	static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }

这个hash方法会算出这个key的hash值。

    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }


putVal方法首先会判断哈希表内部的数组是否已存在，不存在的话会调用resize方法扩大哈希表内部的数组长度。

接下来会使用 (n - 1) & hash 的方式得到数组的索引值，这里的n就是HashMap内部数组的length长度，使用n-1可以避免数组长度跃界。

然后判断这个索引下是否有节点。

没有的话使用newNode方法构造一个新的节点作为开始节点，这个新的节点是起始节点，所以下一个节点为null(next属性为null)。 newNode方法如下， 就是直接构造一个节点，然后赋值给对应数组下标下的项。

    Node<K,V> newNode(int hash, K key, V value, Node<K,V> next) {
        return new Node<>(hash, key, value, next);
    }

有的话执行else下的代码。

else下的代码的意思：

if：如果新的key值和数组上第一个节点的key值和新的key的hash值和第一个节点的hash值或者两者key值equals。 那么说明这个新的key跟第一个节点的key一致，然后把第一个节点的引用赋值给变量e。

else if：如果第一个节点是一个红黑树的节点，那么使用putTreeVal方法将节点的引用赋值给变量e。

else：开始各个节点，如果发现遍历节点的下一个节点为null，那么构造一个新的节点并作为当前遍历节点的next节点。否则判断遍历节点的hash值是否重复，重复的话赋值给变量e。接下来处理哈希值重复的数据。

如果put方法新增了节点，判断判断Map中总的键值对个数是否大于阀值(threshold)，大于的话调用resize方法扩展长度。

resize方法会扩展数组的长度，增大为两倍， 还会扩展threshold为原来的两倍。


#### V get(Object key) ####

** 根据键值，得到这个键值上对应的value **

	public V get(Object key) {
        Node<K,V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }

使用getNode方法得带这个键值上对应的节点， getNode方法的参数是键值的hash值和键值自身。

	final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            if ((e = first.next) != null) {
                if (first instanceof TreeNode)
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        return null;
    }

很明显，这个getNode方法的作用就是根据hash值得到数组上的链表，然后遍历这个链表进行比较，最终拿到对应的值，没找到符合条件的值的话返回null。

这里键值比较的方法如下：

	(k = first.key) == key || (key != null && key.equals(k))
    
** 键值的hash值相等或者键值的hash值不等并且equals方法相等 **

#### int size() ####

** 返回map内部键值对的个数 **

	public int size() {
        return size;
    }
    
#### boolean containsKey(Object key) ####

** 根据键值判断是否存在对应的节点 **

	public boolean containsKey(Object key) {
        return getNode(hash(key), key) != null;
    }

根据键值使用getNode方法得到节点，然后判断节点是否存在

#### boolean containsValue(Object value) ####

** 根据值判断是否存在对应的节点 **

	public boolean containsValue(Object value) {
        Node<K,V>[] tab; V v;
        if ((tab = table) != null && size > 0) {
            for (int i = 0; i < tab.length; ++i) {
                for (Node<K,V> e = tab[i]; e != null; e = e.next) {
                    if ((v = e.value) == value ||
                        (value != null && value.equals(v)))
                        return true;
                }
            }
        }
        return false;
    }

遍历map内部数组和链表，然后判断各个链表上键值对的值是否与参数相等。

判断条件：

	(v = e.value) == value || (value != null && value.equals(v))

判断是根据值的内存地址是否相等或者值的equals方法是否相等。

#### V remove(Object key) ####

** 根据键值移除节点 **

	public V remove(Object key) {
        Node<K,V> e;
        return (e = removeNode(hash(key), key, null, false, true)) == null ?
            null : e.value;
    }

remove方法内部调用removeNode方法。

	final Node<K,V> removeNode(int hash, Object key, Object value,
                               boolean matchValue, boolean movable) {
        Node<K,V>[] tab; Node<K,V> p; int n, index;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (p = tab[index = (n - 1) & hash]) != null) {
            Node<K,V> node = null, e; K k; V v;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                node = p;
            else if ((e = p.next) != null) {
                if (p instanceof TreeNode)
                    node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
                else {
                    do {
                        if (e.hash == hash &&
                            ((k = e.key) == key ||
                             (key != null && key.equals(k)))) {
                            node = e;
                            break;
                        }
                        p = e;
                    } while ((e = e.next) != null);
                }
            }
            if (node != null && (!matchValue || (v = node.value) == value ||
                                 (value != null && value.equals(v)))) {
                if (node instanceof TreeNode)
                    ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
                else if (node == p)
                    tab[index] = node.next;
                else
                    p.next = node.next;
                ++modCount;
                --size;
                afterNodeRemoval(node);
                return node;
            }
        }
        return null;
    }

先根据键的hash值找到对应的链表表头，然后遍历链表找出对应的节点，找到之后处理对应的链表顺序。 size减一，也就是键值对的个数减一。


## HashMap注意的地方 ##

1. HashMap采用的是拉链式哈希表数据结构
2. HashMap内部存储的数据是无序的，这是因为HashMap内部的数组的下表是根据hash值算出来的

