title: Java AtomicInteger原理分析
date: 2016-03-16 20:32:35
tags:
- java
categories:
- java

--------------

Java中的AtomicInteger大家应该很熟悉，它是为了解决多线程访问Integer变量导致结果不正确所设计的一个基于多线程并且支持原子操作的Integer类。

![](http://7x2wh6.com1.z0.glb.clouddn.com/java-AtomicInteger-analysis.jpeg)

<!--more-->

它的使用也非常简单：

	AtomicInteger ai = new AtomicInteger(0);
    ai.addAndGet(5); // 5
    ai.getAndAdd(1); // 5
	ai.get(); // 6
	
AtomicInteger内部有一个变量UnSafe：

	private static final Unsafe unsafe = Unsafe.getUnsafe();
	
Unsafe类是一个可以执行不安全、容易犯错的操作的一个特殊类。虽然Unsafe类中所有方法都是public的，但是这个类只能在一些被信任的代码中使用。Unsafe的源码可以在这里看 -> [UnSafe源码](http://www.docjar.com/html/api/sun/misc/Unsafe.java.html)。


Unsafe类可以执行以下几种操作：

1. 分配内存，释放内存：在方法allocateMemory，reallocateMemory，freeMemory中，有点类似c中的malloc，free方法
2. 可以定位对象的属性在内存中的位置，可以修改对象的属性值。使用objectFieldOffset方法
3. 挂起和恢复线程，被封装在LockSupport类中供使用
4. CAS操作(CompareAndSwap，比较并交换，是一个原子操作)


AtomicInteger中用的就是Unsafe的CAS操作。

Unsafe中的int类型的CAS操作方法：

	public final native boolean compareAndSwapInt(Object o, long offset,
													int expected,
													int x);


参数o就是要进行cas操作的对象，offset参数是内存位置，expected参数就是期望的值，x参数是需要更新到的值。

也就是说，如果我把1这个数字属性更新到2的话，需要这样调用：

	compareAndSwapInt(this, valueOffset, 1, 2)
	
valueOffset字段表示内存位置，可以在AtomicInteger对象中使用unsafe得到：

	static {
      try {
        valueOffset = unsafe.objectFieldOffset
            (AtomicInteger.class.getDeclaredField("value"));
      } catch (Exception ex) { throw new Error(ex); }
    }


AtomicInteger内部使用变量value表示当前的整型值，这个整型变量还是volatile的，表示内存可见性，一个线程修改value之后保证对其他线程的可见性：

	private volatile int value;


AtomicInteger内部还封装了一下CAS，定义了一个compareAndSet方法，只需要2个参数：

	public final boolean compareAndSet(int expect, int update) {
        return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
    }


addAndGet方法，addAndGet方法内部使用一个死循环，先得到当前的值value，然后再把当前的值加一，加完之后使用cas原子操作让当前值加一处理正确。当然cas原子操作不一定是成功的，所以做了一个死循环，当cas操作成功的时候返回数据。这里由于使用了cas原子操作，所以不会出现多线程处理错误的问题。比如线程A得到current为1，线程B也得到current为1；线程A的next值为2，进行cas操作并且成功的时候，将value修改成了2；这个时候线程B也得到next值为2，当进行cas操作的时候由于expected值已经是2，而不是1了；所以cas操作会失败，下一次循环的时候得到的current就变成了2；也就不会出现多线程处理问题了：

	public final int addAndGet(int delta) {
        for (;;) {
            int current = get();
            int next = current + delta;
            if (compareAndSet(current, next))
                return next;
        }
    }

incrementAndGet方法，跟addAndGet方法类似，只不过next值变成了current+1：

	public final int incrementAndGet() {
        for (;;) {
            int current = get();
            int next = current + 1;
            if (compareAndSet(current, next))
                return next;
        }
    }

getAndAdd方法，跟addAndGet方法一样，返回值变成了current：

	public final int getAndAdd(int delta) {
        for (;;) {
            int current = get();
            int next = current + delta;
            if (compareAndSet(current, next))
                return current;
        }
    }

缺点：

虽然AtomicInteger中的cas操作可以实现非阻塞的原子操作，但是会产生ABA问题，

参考资料：

[http://blog.csdn.net/ghsau/article/details/38471987](http://blog.csdn.net/ghsau/article/details/38471987)

[http://blog.csdn.net/aesop_wubo/article/details/7537278](http://blog.csdn.net/aesop_wubo/article/details/7537278)

[http://ifeve.com/sun-misc-unsafe/](http://ifeve.com/sun-misc-unsafe/)

[http://ifeve.com/java-atomic/](http://ifeve.com/java-atomic/)
