title: jdk HashMap工作原理分析
date: 2016-03-29 01:49:58
tags:
- jdk
- map
categories: jdk

----------------


Map是一个映射键和值的对象。类似于Python中的字典。

HashMap为什么会出现呢?

因为数组这种数据结构，虽然遍历简单，但是插入和删除操作复杂，需要移动数组内部的元素；链表这种数据结构，插入和删除操作简单，但是查找复杂，只能一个一个地遍历。

有没有一种新的数据结构，插入数据简单，同时查找也简单？ 这个时候就出现了哈希表这种数据结构。 这是一种折中的方式，插入没链表快，查询没数组快。

wiki上就是这么定义哈希表的：

散列表（Hash table，也叫哈希表），是根据关键字（Key value）而直接访问在内存存储位置的数据结构。也就是说，它通过计算一个关于键值的函数，将所需查询的数据映射到表中一个位置来访问记录，这加快了查找速度。这个映射函数称做散列函数，存放记录的数组称做散列表。


<!--more-->

有几个概念要解释一下：

1. 如果有1个关键字为k，它是通过一种函数f(k)得到散列表的地址，然后把值放到这个地址上。这个函数f就称为散列函数，也叫哈希函数。
2. 对于不同的关键字，得到了同一地址，即k1 != k2，但是f(k1) = f(k2)。这种现象称为冲突，
3. 若对于关键字集合中的任一个关键字，经散列函数映象到地址集合中任何一个地址的概率是相等的，则称此类散列函数为均匀散列函数

散列函数有好几种实现，分别有直接定址法、随机数法、除留余数法等，在[wiki散列表](https://zh.wikipedia.org/wiki/%E5%93%88%E5%B8%8C%E8%A1%A8)上都有介绍。

散列表的冲突解决方法，也有好几种，有开放定址法、单独链表法、再散列等。

Java中的HashMap采用的冲突解决方法是使用单独链表法，如下图所示：

![](http://7x2wh6.com1.z0.glb.clouddn.com/hashmap01.png)

## HashMap原理分析 ##

HashMap是jdk中Map接口的实现类之一，是一个散列表的实现。

HashMap中的key和value都可以为null，且它的方法都没有synchronized。 其他方法的实现大部分跟HashTable一致。HashTable的相关源码不在这里介绍，基本上跟HashTable一致。

HashMap有个内部静态类Node，这个Node就是为了解决冲突而设计的链表中的节点的概念。它有4个属性，hash表示哈希地址，key表示关键字，value表示值, next表示这个节点的下一个节点，是一个单项链表：

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

        ...
    }
    
### 在分析HashMap源码之前，先看一个HashMap使用例子 ###

	Map<String, Integer> map = new HashMap<String, Integer>(5);
	map.put("java", 1);
    map.put("golang", 2);
    map.put("python", 3);
    map.put("ruby", 4);
    map.put("scala", 5);
    
上面这段代码执行之后会生成下面这张哈希表。

![](http://7x2wh6.com1.z0.glb.clouddn.com/hashmap05.jpg)

至于为什么会生成这样的哈希表，会在后面分析源码中讲解。

### HashMap的属性 ###

HashMap的几个重要的属性:

	transient Node<K,V>[] table; // 哈希表数组

    transient int size; // 键值对个数

    int threshold; // 阀值。 值 = 容量 * 加载因子。默认值为12(16(默认容量) * 0.75(默认加载因子))。当哈希表中的键值对个数超过该值时，会进行扩容

    final float loadFactor; // 加载因子，默认是0.75
    
有2个重要的特性影响着HashMap的性能，分别是capacity(容量)和load factor(加载因子)。

其中capacity表示哈希表bucket的数量，HashMap的默认值是16。load factor加载因子表示当一个map填满了达到这个比例之后的bucket时候，和ArrayList一样，将会创建原来HashMap大小的两倍的bucket数组，来重新调整map的大小，并将原来的对象放入新的bucket数组中。这个过程也叫做重哈希。默认的load factor为0.75 。

### HashMap的操作 ###

分析一下HashMapput键值对的过程，是如何找到bucket的，遇到哈希冲突的时候是如何使用链表法的。

#### put操作 ####

	public V put(K key, V value) {
		// 第一个参数就是关键字key的哈希值
        return putVal(hash(key), key, value, false, true);
    }
    
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length; // 哈希表是空的话，重新构建，进行扩容
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null); // 没有hash冲突的话，直接在对应位置上构造一个新的节点即可
        else { // 如果哈希表当前位置上已经有节点的话，说明有hash冲突
            Node<K,V> e; K k;
            // 关键字跟哈希表上的首个节点济宁比较
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            // 如果使用的是红黑树，用红黑树的方式进行处理
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else { // 跟链表进行比较
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) { // 一直遍历链表，直到找到最后一个
                        p.next = newNode(hash, key, value, null); // 构造链表上的新节点
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
            if (e != null) { // 如果找到了节点，说明关键字相同，进行覆盖操作，直接返回旧的关键字的值
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        if (++size > threshold) // 如果目前键值对个数已经超过阀值，重新构建
            resize();
        afterNodeInsertion(evict); // 节点插入以后的钩子方法
        return null;
    }

#### get操作 ####

get操作关键点就是怎么在哈希表上取数据，理解了put操作之后，get方法很容易理解了：

	public V get(Object key) {
        Node<K,V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }

getNode方法就说明了如何取数据：

	final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) { // 如果哈希表容量为0或者关键字没有命中，直接返回null
            if (first.hash == hash &&  // 关键字命中的话比较第一个节点
                ((k = first.key) == key || (key != null && key.equals(k)))) 
                return first;
            if ((e = first.next) != null) {
                if (first instanceof TreeNode) // 以红黑树的方式查找
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                do { // 遍历链表查找
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        return null;
    }


#### hash过程和resize过程分析 ####

hash过程在HashMap里就是一个hash方法：

	static final int hash(Object key) {
        int h;
        // 使用hashCode的值和hashCode的值无符号右移16位做异或操作
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
    
这段代码是什么意思呢？ 我们以文中的那个demo为例，说明"java"这个关键字是如何找到对应bucket的过程。

![](http://7x2wh6.com1.z0.glb.clouddn.com/hashmap06.jpg)

从上图可以看到，hash方法得到的hash值是根据关键字的hashCode的高16位和低16位进行异或操作得到的一个值。

这个值再与哈希表容量-1值进行与操作得到最终的bucket索引值。

	(n - 1) & hash

hashCode的高16位与低16位进行异或操作主要是设计者想了一个顾全大局的方法(综合考虑了速度、作用、质量)来做的。

如果链表的数量大了，HashMap会把哈希表转换成红黑树来进行处理，本文不讨论这部分内容。


现在回过头来看例子，为什么初始化了一个容量为5的HashMap，但是哈希表的容量为8，而且阀值为6？

因为HashMap的构造函数初始化threshold的时候调用了tableSizeFor方法，这个方法会把容量改成2的幂的整数，主要是为了哈希表散列更均匀。

	// 定位bucket索引的最后操作。如果n为奇数，n-1就是偶数，偶数的话转成二进制最后一位是0，相反如果是奇数，最后一位是1，这样产生的索引值将更均匀
	(n - 1) & hash

tableSizeFor方法如下：

	this.threshold = tableSizeFor(initialCapacity);

	// 保证thresold为2的幂
	static final int tableSizeFor(int cap) {
        int n = cap - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }
    
阀值为6是因为之后进行resize操作的时候更新了阀值

	阀值 = 容量 * 加载因子 = 8 * 0.75 = 6
    
    
HashMap的扩容会把原先哈希表的容量扩大两倍。扩大之后，会对节点重新进行处理。

哈希表上的节点的状态有3种，分别是单节点，无节点，链表，扩容对于这3种状态的处理方式如下：

以8节点为原先容量，扩容为16容量讲解。

1. 单节点：由于容量扩大两倍，相当于左移1位。扩容前与00000111[7，n - 1 = 8 - 1]进行与操作。扩容后与00001111[15, n - 1 = 16 - 1]进行与操作。所以最终的结果要是还是在原位置，要么在原位置 +8(+old capacity) 位置
2. 无节点：不处理
3. 链表：遍历各个节点，每个节点的处理方式跟单节点一样，结果分成2种，还在原位置和原位置 +8 位置

单节点处理示意图如下，这么设计的原因就是不需要再次计算hash值，只需要移动位置(+old capacity)即可：

![](http://7x2wh6.com1.z0.glb.clouddn.com/hashmap07.jpg)   

下图是一个HashMap扩容之后的效果图（省去了索引为7橙色链表的虚线，太多线条了）：

![](http://7x2wh6.com1.z0.glb.clouddn.com/hashmap08.jpg)   

哈希表扩容是使用resize方法完成：

	final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        if (oldCap > 0) { // 如果老容量大于0，说明哈希表中已经有数据了，然后进行扩容
            if (oldCap >= MAXIMUM_CAPACITY) { // 超过最大容量的话，不扩容
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY && // 容量加倍
                     oldCap >= DEFAULT_INITIAL_CAPACITY) // 如果老的容量超过默认容量的话
                newThr = oldThr << 1; // 阀值加倍
        }
        else if (oldThr > 0) // 根据thresold初始化数组
            newCap = oldThr;
        else {               // 使用默认配置
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        if (oldTab != null) {
            for (int j = 0; j < oldCap; ++j) { // 扩容之后进行rehash操作
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e; // 单节点扩容
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap); // 红黑树方式处理
                    else { // 链表扩容
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            } 
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
    }

## HashMap注意的地方 ##

1. HashMap底层是个哈希表，使用拉链法解决冲突
2. HashMap内部存储的数据是无序的，这是因为HashMap内部的数组的下表是根据hash值算出来的
3. HashMap允许key为null
4. HashMap不是一个线程安全的类

