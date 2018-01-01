title: jdk ArrayDeque工作原理分析
date: 2016-04-03 00:43:59
tags:
- jdk
- collection
categories: jdk

----------------


ArrayDeque双向队列是jdk中列表的一种实现，支持元素在头和尾这两端进行插入和删除操作。

Deque接口(双向队列)的两个主要实现类是ArrayDeque和LinkedList。

其中ArrayDeque底层使用循环数组实现双向队列，而LinkedList是使用链表实现，之前在[jdk LinkedList工作原理分析](http://fangjian0423.github.io/2016/03/27/jdk_linkedlist/)这篇文章中，已经分析过了LinkedList的实现原理，本文分析ArrayDeque的实现原理。

<!--more-->

## 一个ArrayDeque例子 ##

一段ArrayDeque代码：

	ArrayDeque<String> arrayDeque = new ArrayDeque<String>(4);
    arrayDeque.add("1");
    arrayDeque.add("2");
    arrayDeque.add("3");
    arrayDeque.addFirst("0.5");
    arrayDeque.add("4");

ArrayDeque内部使用的循环数组的容量，当首次进行初始化的时候，最小容量为8，如果超过8，扩大成2的幂。

	// 调用带有容量参数的构造函数后，数组初始化过程
	private void allocateElements(int numElements) {
        int initialCapacity = MIN_INITIAL_CAPACITY; // 最小容量为8
		if (numElements >= initialCapacity) { // 如果要分配的容量大于等于8，扩大成2的幂；否则使用最小容量8
            initialCapacity = numElements;
            initialCapacity |= (initialCapacity >>>  1);
            initialCapacity |= (initialCapacity >>>  2);
            initialCapacity |= (initialCapacity >>>  4);
            initialCapacity |= (initialCapacity >>>  8);
            initialCapacity |= (initialCapacity >>> 16);
            initialCapacity++;

            if (initialCapacity < 0)
                initialCapacity >>>= 1;
        }
        elements = new Object[initialCapacity]; // 构造数组
    }
    
上面例子构造容量为4的数组，但是由于最小容量为8，所以构造的数组的容量是8。

执行过程如下：

![](http://7x2wh6.com1.z0.glb.clouddn.com/arraydeque01.jpg)

## ArrayDeque原理分析 ##

ArrayDeque使用的是循环数组，内部有3个属性，分别是：

	Object[] elements; // 数组
	int head; // 头索引
	int tail; // 尾索引
	
### add操作 ###
	
上面例子使用的add方法，其实内部使用了addLast方法，addLast也就添加数据到双向队列尾端：

	public void addLast(E e) {
        if (e == null)
            throw new NullPointerException();
        elements[tail] = e; // 根据尾索引，添加到尾端
        if ( (tail = (tail + 1) & (elements.length - 1)) == head) // 尾索引+1，如果尾索引和头索引重复了，说明数组满了，进行扩容
            doubleCapacity();
    }
    
    
addFirst方法跟addLast方法相反，添加数据到双向队列头端：

	public void addFirst(E e) {
        if (e == null)
            throw new NullPointerException();
        elements[head = (head - 1) & (elements.length - 1)] = e; // 根据头索引，添加到头端，头索引-1
        if (head == tail) // 如果头索引和尾索引重复了，说明数组满了，进行扩容
            doubleCapacity();            
    }
    
### remove操作 ###

remove操作分别removeFirst和removeLast，removeLast代码如下：

	public E removeLast() {
        E x = pollLast(); // 调用pollLast方法
        if (x == null)
            throw new NoSuchElementException();
        return x;
    }
    
    public E pollLast() {
        int t = (tail - 1) & (elements.length - 1); // 尾索引 -1
        @SuppressWarnings("unchecked")
        E result = (E) elements[t]; // 根据尾索引，得到尾元素
        if (result == null)
            return null;
        elements[t] = null; // 尾元素置空
        tail = t;
        return result;
    }
    
removeFirst方法原理一样，remove头元素。 头索引 +1

![](http://7x2wh6.com1.z0.glb.clouddn.com/arraydeque02.jpg)

### 扩容 ###

ArrayDeque的扩容会把数组容量扩大2倍，同时还会重置头索引和尾索引，头索引置为0，尾索引置为原容量的值。

比如容量为8，扩容为16，头索引变成0，尾索引变成8。


扩容代码如下：

	private void doubleCapacity() {
        assert head == tail;
        int p = head;
        int n = elements.length;
        int r = n - p;
        int newCapacity = n << 1;
        if (newCapacity < 0)
            throw new IllegalStateException("Sorry, deque too big");
        Object[] a = new Object[newCapacity];
        System.arraycopy(elements, p, a, 0, r);
        System.arraycopy(elements, 0, a, r, p);
        elements = a;
        head = 0;  // 头索引重置
        tail = n;  // 尾索引重置
    }

![](http://7x2wh6.com1.z0.glb.clouddn.com/arraydeque03.jpg)

## 其他 ##

Deque接口同时还附带了Stack的功能。

	ArrayDeque<String> stack = new ArrayDeque<String>(4);
    stack.push("1");
    stack.push("2");
    stack.push("3");
    String pop = stack.pop(); // 3
    
push方法内部调用addFirst方法，pop方法内部调用removeFirst方法。

![](http://7x2wh6.com1.z0.glb.clouddn.com/arraydeque04.jpg)

## 注意点 ##

1. ArrayDeque是一个使用循环数组实现的双向队列，LinkedList也是一个双向队列，不过它的底层实现是使用链表
2. ArrayDeque的扩容会把数组容量扩大2倍，同时还会重置头索引和尾索引
3. Deque双向队列接口同时也实现了Stack接口，可以把Deque当成Stack使用，它的速度比java.util.Stack要快，因为Stack底层操作数据会加锁，而Deque不会加锁
4. ArrayDeque不是一个线程安全的类
    
