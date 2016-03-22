title: jdk源码分析之LinkedHashMap
date: 2015-02-21 01:53:23
tags:
- jdk
- map
categories: jdk
description: LinkedHashMap是一种会记录插入顺序的Map，HashMap由于hash函数的关系，它是无序的，而LinkedHashMap则是一种会保存插入顺序的哈希表 ...
----------------

## 前言 ##

LinkedHashMap是一种会记录插入顺序的Map，HashMap由于hash函数的关系，它是无序的，而LinkedHashMap则是一种会保存插入顺序的哈希表。

## LinkedHashMap解析 ##

首先是LinkedHashMap的定义：

    public class LinkedHashMap<K,V>
        extends HashMap<K,V>
            implements Map<K,V>

LinkedHashMap继承HashMap，实现Map接口。

接下来是LinkedHashMap的内部类Entry：

	static class Entry<K,V> extends HashMap.Node<K,V> {
        Entry<K,V> before, after;
        Entry(int hash, K key, V value, Node<K,V> next) {
            super(hash, key, value, next);
        }
    }
    
Entry内部类继承HashMap的内部类Node，HashMap内部类Node也就是哈希表上的数组上的节点。

Node表示单向节点，而这个Entry则表示双向节点。

LinkedHashMap有两个重要的属性：

	transient LinkedHashMap.Entry<K,V> head;

    transient LinkedHashMap.Entry<K,V> tail;
    
head表示插入的第一个节点，tail标志最后一个节点。

LinkedHashMap继承自HashMap，大多数方法是使用HashMap的方法，只是覆盖了几个方法。

比如 **V put(K key, V value)** 方法

put方法还是使用HashMap的put方法，put方法内部调用putVal方法。

putVal方法遇到需要构造一个新的节点的时候会调用newNode方法，HashMap有自己的newNode方法：

	Node<K,V> newNode(int hash, K key, V value, Node<K,V> next) {
        return new Node<>(hash, key, value, next);
    }
    
HashMap构造新节点仅仅是实例化了HashMap内部的Node对象，创建了一个新的节点。

LinkedHashMap覆盖了这个newNode方法：

	Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
        LinkedHashMap.Entry<K,V> p =
            new LinkedHashMap.Entry<K,V>(hash, key, value, e);
        linkNodeLast(p);
        return p;
    }
而LinkedHashMap则是创建了Entry对象，也就是一个双向节点，Entry继承自Node，所以也是一个Node类型。这里构建了一个Entry节点之后调用了linkNodeLast方法。

	private void linkNodeLast(LinkedHashMap.Entry<K,V> p) {
        LinkedHashMap.Entry<K,V> last = tail; // 新插入的节点肯定是最后一个节点
        tail = p;
        if (last == null) // 最后一个节点为null的话表示这是第一次构建节点，因此head也就是刚构建的节点
            head = p;
        else {
            p.before = last;  // 新构建的节点的前一个节点是之前最后一个插入节点
            last.after = p;   // 之前最后一个节点的后一个节点是新构建的节点
        }
    }
    
linkNodeLast方法的作用就是更新head和tail这两个属性，确保键值对的插入顺序。

**boolean containsValue(Object value)**方法也被覆盖了。

HashMap的containsValue方法：

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
    
HashMap遍历键值对的时候是根据数组来遍历。
    
LinkedHashMap的containsValue方法：

	public boolean containsValue(Object value) {
        for (LinkedHashMap.Entry<K,V> e = head; e != null; e = e.after) {
            V v = e.value;
            if (v == value || (value != null && value.equals(v)))
                return true;
        }
        return false;
    }

而LinkedHashMap则是根据head属性来遍历，也就是根据第一个节点来遍历。



** LinkedHashMap内部还有个boolean类型的属性accessOrder，accessOrder为false表示迭代顺序是插入顺序，为true表示迭代顺序是访问顺序 **

插入顺序很简单，就是put方法插入的时候的顺序。

那什么是访问顺序呢？   下面分析一下。

首先是LinkedHashMap的get方法，get方法覆盖了HashMap的get方法。

HashMap的get方法：

	public V get(Object key) {
        Node<K,V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }
    
LinkedHashMap的get方法：

	public V get(Object key) {
        Node<K,V> e;
        if ((e = getNode(hash(key), key)) == null)
            return null;
        if (accessOrder)
            afterNodeAccess(e);
        return e.value;
    }
    
HashMap的get方法仅仅是找到对应的节点，而LinkedHashMap的get方法不但查找节点，而且如果这个LinkedHashMap的迭代顺序是访问顺序，那么会调用afterNodeAccess方法：

	void afterNodeAccess(Node<K,V> e) { // move node to last
        LinkedHashMap.Entry<K,V> last;
        if (accessOrder && (last = tail) != e) {
            LinkedHashMap.Entry<K,V> p =
                (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
            p.after = null;
            if (b == null)
                head = a;
            else
                b.after = a;
            if (a != null)
                a.before = b;
            else
                last = b;
            if (last == null)
                head = p;
            else {
                p.before = last;
                last.after = p;
            }
            tail = p;
            ++modCount;
        }
    }

这个afterNodeAccess方法会调整LinkedHashMap内部的节点顺序，把参数e，也就是get方法获取的那个节点放到了最后一个位置，也就是tail属性。 **被访问的节点放到了最后的位置**， 这就是最近最少使用算法，最近使用的节点放到了双向链表的尾部，使用越多的节点越在双向链表的后面，使用越少的节点放到了双向链表的前面。


这个afterNodeAccess方法还在其他方法中被调用，比如put方法，当map插入数据的时候，如果这个key已经存在，那么使用新的数据覆盖旧的数据，同时把这个操作过的节点放到双向链表的结尾；比如replace方法，使用新值替换旧值的时候，操作过的节点会被放到双向链表的结尾。

accessOrder属性只能在下面这个LinkedHashMap的构造函数中指定。 LinkedHashMap的其他构造函数都会设置accessOrder为false。

	public LinkedHashMap(int initialCapacity,
                         float loadFactor,
                         boolean accessOrder) {
        super(initialCapacity, loadFactor);
        this.accessOrder = accessOrder;
    }
    
** 因此访问顺序可以理解为节点的操作顺序，操作越近操作，越在链表的后方。 **
    
## 总结 ##

1. LinkedHashMap也是一种使用拉链式哈希表的数据结构，只不过添加了两个属性：**链表头部和链表尾部**， 而且**LinkedHashMap内部的链表都是双向链表**。

2. LinkedHashMap继承自HashMap，大多数的方法都是跟HashMap一样的，只不过覆盖了一些方法。

3. LinkedHashMap的accessOrder属性决定哈希表的迭代顺序， **accessOrder为true表示迭代顺序为访问顺序，accessOrder为false表示迭代顺序为插入数据顺序**

