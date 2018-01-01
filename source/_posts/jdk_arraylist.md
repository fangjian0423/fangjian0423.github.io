title: jdk ArrayList工作原理分析
date: 2016-03-27 13:33:17
tags:
- jdk
- collection
categories: jdk

----------------


list是一种有序的集合(an ordered collection), 通常也会被称为序列(sequence)，使用list可以精确地控制每个元素的插入，可以通过索引值找到对应list中的各个项，也可以在list中查询元素。

以前的几段话摘自jdk文档的说明。

其实list就相当于一个动态的数组，也就是链表，普通的数组长度大小都是固定的，而list是一个动态的数组，当list的长度满了，再次插入数据到list当中的时候，list会自动地扩展它的长度。

<!--more-->

## ArrayList源码分析 ##

首先我们先分析一个List接口的实现类之一，也是最常用的ArrayList的源码。

ArrayList底层使用一个数组完成数据的添加，查询，删除，修改。这个数组就是下面提到的elementData。

这里分析的代码是基于jdk1.7的。

ArrayList类的属性如下：

    private static final int DEFAULT_CAPACITY = 10; // 集合的默认容量
	private static final Object[] EMPTY_ELEMENTDATA = {}; // 一个空集合数组，容量为0
    private transient Object[] elementData; // 存储集合数据的数组，默认值为null
	private int size; // ArrayList集合中数组的当前有效长度，比如数组的容量是5，size是1 表示容量为5的数组目前已经有1条记录了，其余4条记录还是为空

接下来看一下ArrayList的构造函数：

ArrayList有3个构造函数，分别是

    public ArrayList(int initialCapacity) { // 带有集合容量参数的构造函数
    	// 调用父类AbstractList的方法构造函数
        super();
        if (initialCapacity < 0) // 如果集合的容量小于0，这明显是个错误数值，直接抛出异常
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        this.elementData = new Object[initialCapacity]; // 初始化elementData属性，确定容量
    }

    public ArrayList() { // 没有参数的构造函数
	    super(); // 调用父类AbstractList的方法构造函数
        this.elementData = EMPTY_ELEMENTDATA; // 让elementData和ArrayList的EMPTY_ELEMENTDATA这个空数组使用同一个引用
    }

    public ArrayList(Collection<? extends E> c) { // 参数是一个集合的构造函数
    	elementData = c.toArray(); // elementData直接使用参数集合内部的数组
        size = elementData.length; // 初始化数组当前有效长度
        // c.toArray方法可能不会返回一个Object[]结果，需要做一层判断。这个一个Java的bug，可以在http://bugs.java.com/bugdatabase/view_bug.do?bug_id=6260652查看
        if (elementData.getClass() != Object[].class)
            elementData = Arrays.copyOf(elementData, size, Object[].class);
    }

接下来挑几个重要的方法讲解一下：

### add(E e) 方法 ###

这个方法的作用就是把 **元素添加到集合的最后面** 

源码：

	public boolean add(E e) {
        ensureCapacityInternal(size + 1);  // 调用ensureCapacityInternal，参数是集合当前的长度。确保集合容量够大，不够的话需要扩容
        elementData[size++] = e; // 数组容量够的话，直接添加元素到数组最后一个位置即可，同时修改集合当前有效长度
        return true;
    }
    
	private void ensureCapacityInternal(int minCapacity) {
        if (elementData == EMPTY_ELEMENTDATA) { // 如果数组是个空数组，说明调用的是无参的构造函数
        	// 如果调用的是无参构造函数，说明数组容量为0，那就需要使用默认容量
            minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
        }

        ensureExplicitCapacity(minCapacity);
    }    
    
    private void ensureExplicitCapacity(int minCapacity) {
        modCount++;

        // 如果集合需要的最小长度比数组容量要大，那么就需要扩容，已经放不下了
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }
    
    private void grow(int minCapacity) { // 扩容的实现
        int oldCapacity = elementData.length;
        int newCapacity = oldCapacity + (oldCapacity >> 1); // 长度扩大1.5倍
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
		// 将数组拷贝到新长度的数组中
        elementData = Arrays.copyOf(elementData, newCapacity);
    }


以下面这段代码讲解一下扩容的机制：

	// 初始化一个容量为5的数组
	ArrayList<Integer> list = new ArrayList<Integer>(5);
    list.add(1);
    list.add(2);
    list.add(3);
    list.add(4);
    list.add(5);
    // 当添加第6个元素的时候，数组进行了扩容，扩容1.5倍(5+5/2=7)
    list.add(6);
    
    
![](http://7x2wh6.com1.z0.glb.clouddn.com/arraylist01.jpg)

上图2个白色的空间就是扩容出来的，添加第6个元素之后，最后一个元素没被设置。

### add(int index, E element) 方法 ###

这个方法的作用是 **在指定位置插入数据**，该方法的缺点就是如果集合数据量很大，移动元素位置将会话费不少时间：

    public void add(int index, E element) {
        rangeCheckForAdd(index); // 检查索引位置的正确的，不能小于0也不能大于数组有效长度

        ensureCapacityInternal(size + 1);  // 扩容检测
        System.arraycopy(elementData, index, elementData, index + 1,
                         size - index); // 移动数组位置，数据量很大的话，性能变差
        elementData[index] = element; // 指定的位置插入数据
        size++; // 数组有效长度+1
    }
    
![](http://7x2wh6.com1.z0.glb.clouddn.com/arraylist03.jpg)

上图就表示要在容量为5的数组中的第4个位置插入6这个元素，会进行3个步骤：

1. 容量为5，再次加入元素，需要扩容，扩容出2个白色的空间
2. 扩容之后，5和4这2个元素都移到后面那个位置上
3. 移动完毕之后空出了第4个位置，插入元素6

### remove(int index) ###

remove方法就是 **移除对应坐标值上的数据**

    public E remove(int index) {
      rangeCheck(index); // 检查索引值是否合法

      modCount++;
      E oldValue = elementData(index); // 得到对应索引位置上的元素

      int numMoved = size - index - 1; // 需要移动的数量
      if (numMoved > 0)
          System.arraycopy(elementData, index+1, elementData, index,
                           numMoved); // 从后往前移，留出最后一个元素
      elementData[--size] = null; // 清楚对应位置上的对象，让gc回收

      return oldValue;
    }

比如要移除5个元素中的第3个元素，首先要把4和5这2个位置的元素分别set到3和4这2个位置上，set完之后最后一个位置也就是第5个位置set为null。

![](http://7x2wh6.com1.z0.glb.clouddn.com/arraylist02.jpg)

### remove(Object o) ###

 **找出数组中的元素，然后移除**

	// 跟remove索引元素一样，这个方法是根据equals比较
    public boolean remove(Object o) {
        if (o == null) {
        	// ArrayList允许元素为null，所以对null值的删除在这个分支里进行
            for (int index = 0; index < size; index++)
                if (elementData[index] == null) {
                    fastRemove(index);
                    return true;
                }
        } else {
        	// 效率比较低，需要从第1个元素开始遍历直到找到equals相等的元素后才进行删除，删除同样需要移动元素
            for (int index = 0; index < size; index++)
                if (o.equals(elementData[index])) {
                    fastRemove(index);
                    return true;
                }
        }
        return false;
    }
    
### clear ###

**清除list中的所有数据**

    public void clear() {
      modCount++;

      // 遍历集合数据，全部set为null
      for (int i = 0; i < size; i++)
          elementData[i] = null;

      size = 0; // 数组有效长度变成0
    }

### set(int index, E element) ###

**用element值替换下标值为index的值**

    public E set(int index, E element) {
      rangeCheck(index); // 检查索引值是否合法

      E oldValue = elementData(index); 
      elementData[index] = element; // 直接替换
      return oldValue;
    }

### get(int index) ###

**得到下标值为index的元素**

    public E get(int index) {
        rangeCheck(index); // 检查索引值是否合法

        return elementData(index); // 直接返回下标值
    }

### addAll ###

**在列表的结尾添加一个Collection集合**

    public boolean addAll(Collection<? extends E> c) {
        Object[] a = c.toArray();
        int numNew = a.length;
        ensureCapacityInternal(size + numNew);  // 扩容检测
        System.arraycopy(a, 0, elementData, size, numNew); // 直接在数组后面添加新的数组中的所有元素
        size += numNew; // 更新有效长度
        return numNew != 0;
    }

### toArray ###

**根据elementData数组拷贝一份新的数组**

    public Object[] toArray() {
        return Arrays.copyOf(elementData, size);
    }

## ArrayList的注意点 ##

1. 当数据量很大的时候，ArrayList内部操作元素的时候会移动位置，很耗性能
2. ArrayList虽然可以自动扩展长度，但是数据量一大，扩展的也多，会造成很多空间的浪费
3. ArrayList有一个内部私有类，SubList。ArrayList提供一个subList方法用于构造这个SubList。这里需要注意的是SubList和ArrayList使用的数据引用是同一个对象，在SubList中操作数据和在ArrayList中操作数据都会影响双方。
4. ArrayList允许加入null元素
