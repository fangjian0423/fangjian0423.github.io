title: jdk源码分析之ArrayList
date: 2015-01-16 21:07:44
tags: [jdk,list,ArrayList]
description: 介绍jdk内部的ArrayList相关的知识
----------------

## 前言 ##

list是一种有序的集合(an ordered collection), 通常也会被称为序列(sequence)，使用list可以精确地控制每个元素的插入，可以通过索引值找到对应list中的各个项，也可以在list中查询元素。

list跟set不同，list允许出现重复的数据，list还允许内部出现null的项。

以前的几段话摘自jdk文档的说明。

其实list就相当于一个动态的数组，也就是链表，普通的数组长度大小都是固定的，而list是一个动态的数组，当list的长度满了，再次插入数据到list当中的时候，list会自动地扩展它的长度。

C，C++中有链表的概念，它们可以使用结构体和指针来定义一个链表。 这样链表插入数据的时候可以加入新的结构体。 但是Java中没有指针这个概念，只能通过其他方式来实现。 比如数组。

## ArrayList源码分析 ##

首先我们先分析一个List接口的实现类之一，也是最常用的ArrayList的源码。

ArrayList其实就是一个单向链表。

这里分析的代码是基于jdk1.8的。

ArrayList类的属性如下：

    private static final Object[] EMPTY_ELEMENTDATA = {};
	  private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
    transient Object[] elementData;
    private int size;

属性size表示这个集合的有效长度；Object[] elementData这个数组是这个集合的全部元素；EMPTY_ELEMENTDATA表示一个共享的空数组，主要是为了让所有的空集合共享使用的；DEFAULTCAPACITY_EMPTY_ELEMENTDATA表示共享的空数组，是为了给有默认大小的空集合使用的。
(transient关键字表示这个属性不会被序列化)

接下来是ArrayList的构造函数分析：

ArrayList有3个构造函数，分别是

    public ArrayList(int initialCapacity) {
      if (initialCapacity > 0) {
        this.elementData = new Object[initialCapacity];
      } else if (initialCapacity == 0) {
        this.elementData = EMPTY_ELEMENTDATA;
      } else {
          throw new IllegalArgumentException("Illegal Capacity: "+
                                              initialCapacity);
      }
    }

简单分析一下，这个构造函数的参数是一个initialCapacity整型类型的参数。这个initialCapacity看名字也知道，就是个初始化容量的意思。 如果这个值比0大，那么内部的elementData数组属性会被初始化；如果这个值等于0，那么内部的elementData会被赋值成EMPTY_ELEMENTDATA，也就是之前分析属性的时候说的让空集合共享使用的一个空数组；如果这个值小于0，抛异常。

    public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }

如果构造ArrayList没有传递任何参数，那么内部的elementData属性会被赋值成DEFAULTCAPACITY_EMPTY_ELEMENTDATA。

    public ArrayList(Collection<? extends E> c) {
        elementData ArrayList= c.toArray();
        if ((size = elementData.length) != 0) {
            // c.toArray might (incorrectly) not return Object[] (see 6260652)
            if (elementData.getClass() != Object[].class)
                elementData = Arrays.copyOf(elementData, size, Object[].class);
        } else {
            // replace with empty array.
            this.elementData = EMPTY_ELEMENTDATA;
        }
    }

这个构造函数的参数是一个集合接口类型。如果这个集合内部的元素大于0，那么这些元素会被拷贝到ArrayList内部的elementData属性中，否则elementData属性使用共享的空数组。

接下来我们比较一下jdk1.6的ArrayList。

jdk1.6的ArrayList只有两个属性：

    private transient Object[] elementData;
    private int size;

它的构造函数：

    public ArrayList(int initialCapacity) {
      super();
      if (initialCapacity < 0)
      throw new IllegalArgumentException("Illegal Capacity: "
      + initialCapacity);
      this.elementData = new Object[initialCapacity];
    }

这个构造函数跟1.8版本的区别不大，就是空list的时候，1.8版本的把共享的空数组赋值给elementData。

    public ArrayList() {
      this(10);
    }

这个构造函数跟1.8版本的相差就大了。他会在内部默认初始化一个10个长度的数组，而1.8版本的是使用一个空的数组赋值给elementData。

    public ArrayList(Collection<? extends E> c) {
      elementData = c.toArray();
      size = elementData.length;
      // c.toArray might (incorrectly) not return Object[] (see 6260652)
      if (elementData.getClass() != Object[].class)
      elementData = Arrays.copyOf(elementData, size, Object[].class);
    }

这个构造函数跟1.8版本的差别是1.8版本下如果这个Collection参数中元素个数为0,那么elementData会被赋值成共享的空数组。

接下来挑几个重要的方法讲解一下：

### add(E e) 方法 ###

这个方法的作用就是把 **元素添加到列表的最后面** ，我先把add方法的结论讲一下(由于ArrayList是一个动态的数组，当添加元素当数组的时候，肯定需要判断一下数组的长度是否够长，不够长的话会做一些处理)：

我觉得可以使用下面这个公式来表示ArrayList内部数组长度的增加算法：

**新的长度 = max(数组长度 + 数组长度 / 2, 新的长度)**

这个算法有个前提条件：**新的长度 > 数组长度**

**这里新的长度不固定，比如add方法: 这个新的长度的就是当前数组有效长度+1; 比如addAll方法: 这个新的长度就是当前数组有效长度+Collection集合长度**

**这里的有效数组长度指的是ArrayList内部的size属性。**

源码：

add方法jdk1.8版本：

    public boolean add(E e) {
      ensureCapacityInternal(size + 1);  // Increments modCount!!
      elementData[size++] = e;
      return true;
    }

添加元素到列表的最后，这里的ensureCapacityInternal方法会处理数组长度不够长的问题，它的参数是当前数组的长度+1。

    private void ensureCapacityInternal(int minCapacity) {
      if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
      }

      ensureExplicitCapacity(minCapacity);
    }

如果当前对象是使用无參的构造函数实现的，那么取10(DEFAULT_CAPACITY)和参数的较大值，然后用这个值调用ensureExplicitCapacity方法。否则，直接调用ensureExplicitCapacity方法。

    private void ensureExplicitCapacity(int minCapacity) {
      modCount++;

      // overflow-conscious code
      if (minCapacity - elementData.length > 0)
        grow(minCapacity);
    }

如果要加的元素的下标值比当前数组长度要大的话(minCapacity参数代表当前数组的最后一个元素的下标值， 只有一个情况特殊，那就是使用无參的构造函数构造之后第一次调用，这个时候minCapacity为10)

    private void grow(int minCapacity) {
      // overflow-conscious code
      int oldCapacity = elementData.length;
      int newCapacity = oldCapacity + (oldCapacity >> 1);
      if (newCapacity - minCapacity < 0)
      newCapacity = minCapacity;
      if (newCapacity - MAX_ARRAY_SIZE > 0)
      newCapacity = hugeCapacity(minCapacity);
      // minCapacity is usually close to size, so this is a win:
      elementData = Arrays.copyOf(elementData, newCapacity);
    }

这个就是数组长度动态计算的算法。

下面是我写的一些测试代码和注释，有兴趣的可以验证一下：

    @Test
    public void testNoParamConstructor() {
      // 不带参数的构造函数第一次add之后，会分配10个长度给内部elementData数组属性
      // 接下来每次add，会都判断数组长度是否不够，不够的话 新的长度 = max(当前数组长度 + 当前数组长度 / 2, 当前数组长度 + 1)
      // 比如，实例化之后。第一次add，数组长度分配了10个空间，当添加第11个的时候，数组长度不够，这也时候会分配15个(11 + 11 / 2)和11个的较大值，然后分配了11个。
      // 然后elementData有15个空间，当添加第16个的时候，分配22(15 + 15 / 2)个。 以此类推， 22个不够的时候分配33个，33个不够的时候分配49个。
      ArrayList list = new ArrayList();

      int count = 164;
      for (int i = 1; i <= count; i++) {
        list.add(i);
      }
      System.out.println("done");
    }

    @Test
    public void testKnowSize() {
      // 带长度的原理。 只不过一开始不会分配10个空间，而是参数穿进来的空间大小。
      // 比如传入0, 0个空间，第一次add, 选择0(0 + 0 / 2)和1(0+1)的较大值，取1,分配1个空间
      // 再次添加的时候，选择1(1+1/2)和2(1+1)的较大值，取2,分配2个空间
      // 再次添加的时候，选择3(2+2/2)和3(2+1)的较大值，取3,分配3个空间
      // 再次添加的时候，选择4(3+3/2)和4(3+1)的较大值，取4,分配4个空间
      // 再次添加的时候，选择6(4+4/2)和5(4+1)的较大值，取6,分配6个空间  以此类推...
      ArrayList list = new ArrayList(0);
      list.add(1);
      list.add(2);
      list.add(3);
      list.add(4);
      list.add(5);
      System.out.println("done");
    }

    @Test
    public void testValidAdd() {
      // 虽然list内部的elementData长度为5,但是有效长度为0。 所以add的时候不会增加长度。
      ArrayList list = new ArrayList(5);
      list.add("1");
    }


### add(int index, E element) 方法 ###

这个方法的作用是 **在指定位置插入数据**

    public void add(int index, E element) {
      rangeCheckForAdd(index);

      ensureCapacityInternal(size + 1);  // Increments modCount!!
      System.arraycopy(elementData, index, elementData, index + 1,
                       size - index);
        elementData[index] = element;
        size++;
    }

rangeCheckForAdd方法是验证参数index是否ok，index < 0 或 index > 数组最后一个下标值的话，直接抛出IndexOutOfBoundsException异常。ensureCapacityInternal方法之前分析add(E e)的时候已经分析过了。接下来的一句代码就是将index下标值的数据全部往后移一位，然后index下标值的数据使用新的素质，元素长度+1(size++)。

### remove(int index) ###

remove方法就是 **移除对应坐标值上的数据**

    public E remove(int index) {
      rangeCheck(index);

      modCount++;
      E oldValue = elementData(index);

      int numMoved = size - index - 1;
      if (numMoved > 0)
          System.arraycopy(elementData, index+1, elementData, index,
                           numMoved);
      elementData[--size] = null; // clear to let GC do its work

      return oldValue;
    }

首先先检查坐标值是否合法，然后得到对应坐标上的元素，之后index下标值之后的数据全部往前移。

测试代码：

    @Test
    public void testRemove() {
      ArrayList list = new ArrayList();
      list.add("1");
      list.add("2");
      list.add("1");
      list.add("3");
      list.remove(0);
      // remove方法只会删除第一个匹配的元素，之后的元素下标都向前移。 所以"1"的下标值从2变成了1，且只有1个"1"元素
      System.out.println(list.indexOf("1")); // 输出 1
    }

### remove(Object o) ###

 **找出数组中的元素，然后移除**

    public boolean remove(Object o) {
        if (o == null) {
            for (int index = 0; index < size; index++)
                if (elementData[index] == null) {
                    fastRemove(index) {
        modCount++;
        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work
    }

### clear ###

**清除list中的所有数据**。 没啥好说的，遍历数组，每一项设置为null，然后让GC去回收。 之后将size设置为0。

    public void clear() {
      modCount++;

      // clear to let GC do its work
      for (int i = 0; i < size; i++)
          elementData[i] = null;

      size = 0;
    }

### set(int index, E element) ###

**用element值替换下标值为index的值**

    public E set(int index, E element) {
      rangeCheck(index);

      E oldValue = elementData(index);
      elementData[index] = element;
      return oldValue;
    }

### get(int index) ###

**得到下标值为index的元素**

    public E get(int index) {
        rangeCheck(index);

        return elementData(index);
    }

### indexOf(Object) ###

**根据参数对象，如果是null对象，找出数组上第一个为null对着的元素，然后返回下标值。如果不是null对象，参数对象equals数组元素为true的话，返回对应的下标**

    public int indexOf(Object o) {
        if (o == null) {
            for (int i = 0; i < size; i++)
                if (elementData[i]==null)
                  return i;
        } else {
            for (int i = 0; i < size; i++)
                if (o.equals(elementData[i]))
                    return i;
        }
        return -1;
    }
  

### contains(Object o) ###

**查看数组中是否存在对象** 内部使用indexOf方法

    public boolean contains(Object o) {
        return indexOf(o) >= 0;
    }

### addAll ###

**在列表的结尾添加一个Collection集合**

    public boolean addAll(Collection<? extends E> c) {
        Object[] a = c.toArray();
        int numNew = a.length;
        ensureCapacityInternal(size + numNew);  // Increments modCount
        System.arraycopy(a, 0, elementData, size, numNew);
        size += numNew;
        return numNew != 0;
    }

首先先得到Collection参数内部数组的长度，然后调用ensureCapacityInternal方法确保内部数据的长度，然后在内部数组的最后添加新的数组。


在列表的指定位置添加一个Collection集合。

    public boolean addAll(int index, Collection<? extends E> c) {
      rangeCheckForAdd(index);

      Object[] a = c.toArray();
      int numNew = a.length;
      ensureCapacityInternal(size + numNew);  // Increments modCount

      int numMoved = size - index;
      if (numMoved > 0)
      System.arraycopy(elementData, index, elementData, index + numNew,
        numMoved);

        System.arraycopy(a, 0, elementData, index, numNew);
        size += numNew;
        return numNew != 0;
    }

首先先判断下标值是否合法。然后调用ensureCapacityInternal方法确保内部数据的长度，然后在index坐标位置的元素都往后移参数Collection集合内部数组的长度，然后空闲的位置加入新集合的数据。

### toArray ###

**根据elementData数组拷贝一份新的数组**

    public Object[] toArray() {
        return Arrays.copyOf(elementData, size);
    }


以前分析的一些方法都会处理一个叫做modCount的int类型的属性，这个属性的ArrayList的父类AbstractList中定义的。

据说这个modCount跟ArrayList的迭代器有关，现在还不清楚。。 以后看迭代器相关的源码的时候我再继续这个话题。


## ArrayList的注意点 ##

1. 当数据量很大的时候，ArrayList内部操作元素的时候会移动位置，很耗性能
2. ArrayList虽然可以自动扩展长度，但是数据量一大，扩展的也多，会造成很多空间的浪费
3. ArrayList有一个内部私有类，SubList。ArrayList提供一个subList方法用于构造这个SubList。这里需要注意的是SubList和ArrayList使用的数据引用是同一个对象，在SubList中操作数据和在ArrayList中操作数据都会影响对方。
4. ArrayList允许加入null元素
