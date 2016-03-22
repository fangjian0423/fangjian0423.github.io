title: jdk源码分析之LinkedList
date: 2015-01-17 16:43:37
tags:
- jdk
- list
categories:
- jdk
description: ArrayList是一个单向链表，LinkedList则是一个双向链表 ...
----------------

## 前言 ##

ArrayList是一个单向链表，LinkedList则是一个双向链表。

双向链表的设计跟单项链表不一样，单项链表只是在内部维护一个数组属性即可。

双向链表不使用数组这个概念，而是在内部定义了一个叫做Node类型的内部类，这个Node就是一个节点，这个节点有3个属性，分别是元素(当前节点要表示的值), 前节点(当前节点之前位置上的一个节点)，后节点(当前节点后面位置的一个节点)。 

LinkedList关于数据的插入，删除操作都会处理这些节点的前后关系。

这是两者最大的区别。

## 源码分析 ##

在分析LinkedList之前，我们先看下它里面的内部类Node，也就是节点类：

	private static class Node<E> {
        E item;
        Node<E> next;
        Node<E> prev;

        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
    }
   
构造函数有3个参数，分别是前一个节点，元素值，后一个节点。

3个属性，元素代表的值，前一个节点，后一个节点。

LinkedList的3个属性：

	transient int size = 0;

    transient Node<E> first;

    transient Node<E> last;
    
size表示这个列表当前的长度, first属性和last属性都是一个Node类型的实例，也就是一个节点。 看名字也知道，一个是第一个节点，另外一个是最后一个节点。

### add(E e) ###

** 添加元素到列表的最后一个位置 **

	public boolean add(E e) {
        linkLast(e);
        return true;
    }
    
add方法内部调用linkLast方法。

	void linkLast(E e) {
        final Node<E> l = last;
        final Node<E> newNode = new Node<>(l, e, null);
        last = newNode;
        if (l == null)
            first = newNode;
        else
            l.next = newNode;
        size++;
        modCount++;
    }

首先先拿到列表中的最后一个节点，然后构造一个新的节点，这个新的节点的前一个元素就是刚才拿到的列表中的最后一个节点，然后属性last也就是最后一个节点赋值成新构造的这个节点，接下来做了一个判断，如果当前列表的最后一个节点为null，那么说明这个列表当前没有元素，于是这个新构造的节点赋值给了第一个节点。否则的话，最后一个节点的下一个节点就是这个新构造的节点。之后列表长度+1(size++)。

总体来说，这个add方法的速度很快，几乎不消耗时间。

### add(int index, E element) ###

** 添加元素到列表中的指定位置 **

	public void add(int index, E element) {
        checkPositionIndex(index);

        if (index == size)
            linkLast(element);
        else
            linkBefore(element, node(index));
    }

首先会判断当前的索引位置是否合法。接下来判断参数索引是否跟列表长度相等， 相等的话说明这个位置就是最后一个位置，那么执行linkLast方法，这个方法之前刚分析过。 如果不是最后一个位置，那么执行linkBefore方法。先看下执行linkBefore方法传入参数的时候调用的一个node方法。

	Node<E> node(int index) {
        // assert isElementIndex(index);

        if (index < (size >> 1)) {
            Node<E> x = first;
            for (int i = 0; i < index; i++)
                x = x.next;
            return x;
        } else {
            Node<E> x = last;
            for (int i = size - 1; i > index; i--)
                x = x.prev;
            return x;
        }
    }
    
node(index)方法的作用是根据索引值，得到该索引下的那个节点。

这个使用了一个小算法：**如果索引值比当前列表长度的一半还要小，那么从第一个元素开始遍历，找到index位置上的元素；否则从最后一个元素开始遍历，找到index位置上的元素**

	void linkBefore(E e, Node<E> succ) {
        // assert succ != null;
        final Node<E> pred = succ.prev;
        final Node<E> newNode = new Node<>(pred, e, succ);
        succ.prev = newNode;
        if (pred == null)
            first = newNode;
        else
            pred.next = newNode;
        size++;
        modCount++;
    }    
    
这个方法的参数e就是新节点的元素值，succ参数是要新插入的节点的那个位置上的现有节点。

处理过程：没啥好说的，处理的就是节点位置的一系列变化。

addAll方法也就是处理一些位置的变化，有兴趣的读者可自行查看源码。

LinkedList还提供了2个不一样的add方法，分别是addFirst和addLast方法，作用分别是插入第一个节点和插入最后一个节点。

### remove(int index) ###

** 移除指定位置上的节点 **

	public E remove(int index) {
        checkElementIndex(index);
        return unlink(node(index));
    }

首先会检查下标值的合法性，然后调用unlink方法:

	E unlink(Node<E> x) {
        // assert x != null;
        final E element = x.item;
        final Node<E> next = x.next;
        final Node<E> prev = x.prev;

        if (prev == null) {
            first = next;
        } else {
            prev.next = next;
            x.prev = null;
        }

        if (next == null) {
            last = prev;
        } else {
            next.prev = prev;
            x.next = null;
        }

        x.item = null;
        size--;
        modCount++;
        return element;
    }

把要移除的节点的前后节点处理一下。移除节点的前一个节点的后一个节点是移除节点的后一个节点。移除节点的后一个节点的前一个节点是移除节点的前一个节点。然后判断一下是否是第一个节点的问题，之后将移除节点上的前后节点的属性值和它的元素值都设置为null。列表长度-1(size--)。

### get(int index) ###

** 得到索引位置上的元素 **

	public E get(int index) {
        checkElementIndex(index);
        return node(index).item;
    }

找到对应的节点，拿出元素值即可。

先讲这么多吧，其他方法就不一一看了。数据结构了解就行了。

## LinkedList和ArrayList的比较  ##

1. LinkedList和ArrayList的设计理念完全不一样，ArrayList基于数组，而LinkedList基于节点。
2. 两者的使用场景不同，ArrayList适用于频繁遍历，但是不怎么操作数据的场合。LinkedList适用于频繁操作数据，不频繁遍历的场合。 刚好相反。 为什么说LinkedList不适合频繁遍历，那是因为LinkedList要遍历节点的话只能从第一个或最后一个节点开始遍历，一个一个找。而ArrayList完全不需要，因为ArrayList内部维护着一个数组，遍历的话仅仅从数组上的索引开始找即可。
3. 两者的数据结构不同，ArrayList是一个单项的链表，而LinkedList是一个双向的链表。

## 使用LinkedList要注意的点 ##

LinkedList在频繁遍历数据的场合下不适合使用。
