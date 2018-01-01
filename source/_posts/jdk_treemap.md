title: jdk TreeMap工作原理分析
date: 2016-04-07 00:55:31
tags:
- jdk
- map
categories: jdk

----------------

TreeMap是jdk中基于红黑树的一种map实现。HashMap底层是使用链表法解决冲突的哈希表，LinkedHashMap继承自HashMap，内部同样也是使用链表法解决冲突的哈希表，但是额外添加了一个双向链表用于处理元素的插入顺序或访问访问。

既然TreeMap底层使用的是红黑树，首先先来简单了解一下红黑树的定义。

红黑树是一棵平衡二叉查找树，同时还需要满足以下5个规则：

1. 每个节点只能是红色或者黑点
2. 根节点是黑点
3. 叶子节点(Nil节点，空节点)是黑色节点
4. 如果一个节点是红色节点，那么它的两个子节点必须是黑色节点(一条路径上不能出现相邻的两个红色节点)
5. 从任一节点到其每个叶子节点的所有路径都包含相同数目的黑色节点

红黑树的这些特性决定了它的查询、插入、删除操作的时间复杂度均为O(log n)。

<!--more-->


## 一个TreeMap例子 ##

一段TreeMap代码：

	TreeMap<Integer, String> treeMap = new TreeMap<Integer, String>();
    treeMap.put(1, "语文");
    treeMap.put(2, "数学");
    treeMap.put(3, "英语");
    treeMap.put(4, "政治");
    treeMap.put(5, "物理");
    treeMap.put(6, "化学");
    treeMap.put(7, "生物");
    treeMap.put(8, "体育");

执行过程：

![](http://7x2wh6.com1.z0.glb.clouddn.com/treemap01.jpg?imageView2/1/w/10000/h/10000/q/100|watermark/2/text/ZmFuZ2ppYW4wNDIzLmdpdGh1Yi5pbw==/font/5a6L5L2T/fontsize/500/fill/I0VGRUZFRg==/dissolve/100/gravity/SouthEast/dx/10/dy/10)

从上面这个例子看到，红黑树添加新节点的时候可能会对节点进行旋转，以保证树的局部平衡。

## TreeMap原理分析 ##

TreeMap内部类Entry表示红黑树中的节点：

	static final class Entry<K,V> implements Map.Entry<K,V> {
        K key; // 关键字
        V value; // 值
        Entry<K,V> left; // 左节点
        Entry<K,V> right; // 右节点
        Entry<K,V> parent; // 父节点
        boolean color = BLACK; // 颜色，默认为黑色

        Entry(K key, V value, Entry<K,V> parent) {
            this.key = key;
            this.value = value;
            this.parent = parent;
        }

        ...
    }

TreeMap的属性：

	private transient Entry<K,V> root; // 根节点
	
    private transient int size = 0; // 节点个数
    
### put操作 ###

红黑树新节点的添加一定是红色节点，添加完新的节点之后会进行旋转操作以保持红黑树的特性。

为什么新添加的节点一定是红色节点，如果添加的是黑色节点，那么就会导致根到叶子的路径上有一条路上，多一个额外的黑节点，这个是很难调整的；但是如果插入的是红色节点，只需要解决其父节点也为红色节点的这个冲突即可。


以N为新插入节点，P为其父节点，U为其父节点的兄弟节点，R为P和U的父亲节点进行分析。如果N的父节点为黑色节点，那直接添加新节点即可，没有产生冲突。如果出现P节点是红色节点，那便产生冲突，可以分为以下几种冲突：

(1) P为红色节点，且U也为红色节点，P不论是R的左节点还是右节点

![](http://7x2wh6.com1.z0.glb.clouddn.com/treemap04.jpg?imageView2/1/w/10000/h/10000/q/100|watermark/2/text/ZmFuZ2ppYW4wNDIzLmdpdGh1Yi5pbw==/font/5a6L5L2T/fontsize/500/fill/I0VGRUZFRg==/dissolve/100/gravity/SouthEast/dx/10/dy/10)

将P和U接口变成黑色节点，R节点变成红色节点。修改之后如果R节点的父节点也是红色节点，那么在R节点上执行相同操作，形成了一个递归过程。如果R节点是根节点的话，那么直接把R节点修改成黑色节点。


(2) P为红色节点，U为黑色节点或缺少，且N是P的右节点、P是R的左节点 或者 P为红色节点，U为黑色节点或缺少，且N是P的左节点、P是R的右节点

![](http://7x2wh6.com1.z0.glb.clouddn.com/treemap07.jpg?imageView2/1/w/10000/h/10000/q/100|watermark/2/text/ZmFuZ2ppYW4wNDIzLmdpdGh1Yi5pbw==/font/5a6L5L2T/fontsize/500/fill/I0VGRUZFRg==/dissolve/100/gravity/SouthEast/dx/10/dy/10)

这两种情况分别对P进行左旋和右旋操作。操作结果就变成了冲突3。 (总结起来就是左右变左左，右左变右右)

(3) P为红色节点，U为黑色节点或缺少，且N是P的左节点、P是R的左节点 或者 P为红色节点，U为黑色节点或缺少，且N是P的右节点、P是R的右节点

![](http://7x2wh6.com1.z0.glb.clouddn.com/treemap08.jpg?imageView2/1/w/10000/h/10000/q/100|watermark/2/text/ZmFuZ2ppYW4wNDIzLmdpdGh1Yi5pbw==/font/5a6L5L2T/fontsize/500/fill/I0VGRUZFRg==/dissolve/100/gravity/SouthEast/dx/10/dy/10)

这两种情况分别对祖父R进行右旋和左旋操作。完美解决冲突。(总结起来就是左左祖右，右右祖左)

这3个新增节点的冲突处理方法了解之后，我们回过头来看本文一开始的例子中添加最后一个[8:体育]节点是如何处理冲突的：

![](http://7x2wh6.com1.z0.glb.clouddn.com/treemap09.jpg?imageView2/1/w/10000/h/10000/q/100|watermark/2/text/ZmFuZ2ppYW4wNDIzLmdpdGh1Yi5pbw==/font/5a6L5L2T/fontsize/500/fill/I0VGRUZFRg==/dissolve/100/gravity/SouthEast/dx/10/dy/10)

接下来我们看TreeMap是如何实现新增节点并处理冲突的。

TreeMap对应的put方法：

	public V put(K key, V value) {
        Entry<K,V> t = root;
        if (t == null) { // 如果根节点是空的，说明是第一次插入数据
            compare(key, key);

            root = new Entry<>(key, value, null); // 构造根节点，并赋值给属性root，默认颜色是黑色
            size = 1; // 节点数 = 1
            modCount++;
            return null;
        }
        int cmp;
        Entry<K,V> parent;
        // split comparator and comparable paths
        Comparator<? super K> cpr = comparator;
        if (cpr != null) { // 比较器存在
            do { // 遍历寻找节点，关键字比节点小找左节点，比节点大的找右节点，直到找到那个叶子节点，会保存需要新构造节点的父节点到parent变量里
                parent = t;
                cmp = cpr.compare(key, t.key);
                if (cmp < 0)
                    t = t.left;
                else if (cmp > 0)
                    t = t.right;
                else
                    return t.setValue(value); // 关键字存在的话，直接用值覆盖原节点的关键字的值，并返回
            } while (t != null);
        }
        else { // 比较器不存在
            if (key == null)
                throw new NullPointerException();
            @SuppressWarnings("unchecked")
                Comparable<? super K> k = (Comparable<? super K>) key; // 比较器不存在直接将关键字转换成比较器，如果关键字不是一个Comparable接口实现类，将会报错
            do { // 遍历寻找节点，关键字比节点小找左节点，比节点大的找右节点，直到找到那个叶子节点，会保存需要新构造节点的父节点到parent变量里
                parent = t;
                cmp = k.compareTo(t.key);
                if (cmp < 0)
                    t = t.left;
                else if (cmp > 0)
                    t = t.right;
                else
                    return t.setValue(value); // 关键字存在的话，直接用值覆盖原节点的关键字的值，并返回
            } while (t != null);
        }
        Entry<K,V> e = new Entry<>(key, value, parent); // 构造新的关键字节点
        if (cmp < 0) // 需要在左节点构造
            parent.left = e;
        else // 需要在右节点构造
            parent.right = e;
        fixAfterInsertion(e); // 插入节点之后，处理冲突以保持树符合红黑树的特性
        size++;
        modCount++;
        return null;
    }
    

fixAfterInsertion方法处理红黑树冲突实现如下：
    
    private void fixAfterInsertion(Entry<K,V> x) {
        x.color = RED; // 新增的节点一定是红色节点

        while (x != null && x != root && x.parent.color == RED) { // P节点是红色节点并且N节点不是根节点的话一直循环
            if (parentOf(x) == leftOf(parentOf(parentOf(x)))) { // P节点是R节点的左节点
                Entry<K,V> y = rightOf(parentOf(parentOf(x))); // y就是U节点
                if (colorOf(y) == RED) { // 如果U节点是红色节点，说明P和U这两个节点都是红色节点，满足冲突(1)
                    setColor(parentOf(x), BLACK); // 冲突(1)解决方案 把P设置为黑色
                    setColor(y, BLACK); // 冲突(1)解决方案 把U设置为黑色
                    setColor(parentOf(parentOf(x)), RED); // 冲突(1)解决方案 把R设置为红色
                    x = parentOf(parentOf(x)); // 递归处理R节点
                } else { // 如果U节点是黑色节点，满足冲突(2)或(3)
                    if (x == rightOf(parentOf(x))) { // 如果N节点是P节点的右节点，满足冲突(2)的第一种情况
                        x = parentOf(x);
                        rotateLeft(x); // P节点进行左旋操作
                    }
                    // P节点左旋操作之后，满足了冲突(3)的第一种情况或者N一开始就是P节点的左节点，这本来就是冲突(3)的第一种情况
                    setColor(parentOf(x), BLACK);  // P节点和R节点交换颜色，P节点变成黑色
                    setColor(parentOf(parentOf(x)), RED); // P节点和R节点交换颜色，R节点变成红色
                    rotateRight(parentOf(parentOf(x))); // R节点右旋操作
                }
            } else { // P节点是R节点的右节点
                Entry<K,V> y = leftOf(parentOf(parentOf(x))); // y就是U节点
                if (colorOf(y) == RED) { // 如果U节点是红色节点，说明P和U这两个节点都是红色节点，满足冲突(1)
                    setColor(parentOf(x), BLACK); // 冲突(1)解决方案 把P设置为黑色
                    setColor(y, BLACK); // 冲突(1)解决方案 把U设置为黑色
                    setColor(parentOf(parentOf(x)), RED); // 冲突(1)解决方案 把R设置为红色
                    x = parentOf(parentOf(x)); // 递归处理R节点
                } else { // 如果U节点是黑色节点，满足冲突(2)或(3)
                    if (x == leftOf(parentOf(x))) { // 如果N节点是P节点的左节点，满足冲突(2)的第二种情况
                        x = parentOf(x);
                        rotateRight(x); // P节点右旋
                    }
                    // P节点右旋操作之后，满足了冲突(3)的第二种情况或者N一开始就是P节点的右节点，这本来就是冲突(3)的第二种情况
                    setColor(parentOf(x), BLACK); // P节点和R节点交换颜色，P节点变成黑色
                    setColor(parentOf(parentOf(x)), RED); // P节点和R节点交换颜色，R节点变成红色
                    rotateLeft(parentOf(parentOf(x))); // R节点左旋操作
                }
            }
        }
        root.color = BLACK; // 根节点是黑色节点
    }

fixAfterInsertion方法的代码跟之前分析的冲突解决方案一模一样。

### get操作 ###

红黑树的get操作相比add操作简单不少，只需要比较关键字即可，要查找的关键字比节点关键字要小的话找左节点，否则找右节点，一直递归操作，直到找到或找不到。代码如下：

	final Entry<K,V> getEntry(Object key) {
        if (comparator != null)
            return getEntryUsingComparator(key);
        if (key == null)
            throw new NullPointerException();
        @SuppressWarnings("unchecked")
            Comparable<? super K> k = (Comparable<? super K>) key;
        Entry<K,V> p = root;
        while (p != null) {
            int cmp = k.compareTo(p.key); // 得到比较值
            if (cmp < 0) // 小的话找左节点
                p = p.left;
            else if (cmp > 0) // 大的话找右节点
                p = p.right;
            else
                return p;
        }
        return null;
    }

### remove操作 ###


红黑树的删除节点跟添加节点一样，比较复杂，删除节点也会让树不符合红黑树的特性，也需要解决这些冲突。

删除操作分为2个步骤：

1. 将红黑树当作一颗二叉查找树，将节点删除
2. 通过"旋转和重新着色"等一系列来修正该树，使之重新成为一棵红黑树

步骤1的删除操作可分为几种情况：

1. 删除节点没有儿子：直接删除该节点
2. 删除节点有1个儿子：删除该节点，并用该节点的儿子节点顶替它的位置
3. 删除节点有2个儿子：可以转成成删除节点只有1个儿子的情况，跟二叉查找树一样，找出节点的右子树的最小元素(或者左子树的最大元素，这种节点称为后继节点)，并把它的值转移到删除节点，然后删除这个后继节点。这个后继节点最多只有1个子节点(如果有2个子节点，说明还能找出右子树更小的值)，所以这样删除2个儿子的节点就演变成了删除没有儿子的节点和删除只有1个儿子的节点的情况

删除节点之后要考虑的问题就是红黑树失衡的调整问题。

步骤2遇到的调整问题只有2种情况：

1. 删除节点没有儿子节点
2. 删除节点只有1个儿子节点


删除节点没有儿子节点的话，直接把节点删除即可。如果节点是黑色节点，需要进行平衡性调整，否则，不用调整平衡性。这里的平衡性调整跟删除只有1个儿子节点一样，删除只有1个儿子的调整会先把节点删除，然后儿子节点顶上来，顶上来之后再进行平衡性调整。而删除没有儿子节点的节点的话，先进行调整，调整之后再把这个节点删除。他们的调整策略是一样的，只不过没有儿子节点的情况下先进行调整，然后再删除节点，而有儿子节点的情况下，先把节点删除，删除之后儿子节点顶上来，然后再做平衡性调整。

删除节点只有1个儿子节点还分几种情况：

1. 如果被删除的节点是红色节点，那说明它的父节点是黑色节点，儿子节点也是黑色节点，那么删除这个节点就不会影响红黑树的属性，直接使用它的黑色子节点代替它即可
2. 如果被删除的节点是黑色节点，而它的儿子节点是红色节点。删除这个黑色节点之后，它的红色儿子节点顶替之后，会破坏性质5，只需要把儿子节点重绘为黑色节点即可，这样原先通过黑色删除节点的所有路径被现在的重绘后的儿子节点所代替
3. 如果被删除的节点是黑色节点，而它的儿子节点也是黑色节点。这是一种复杂的情况，因为路径路过被删除节点的黑色节点路径少了1个，导致违反了性质5，所以需要对红黑树进行平衡调整。可分为以下几种情况进行调整：

以N为删除节点的儿子节点(删除之后，处于新的位置上)，它的兄弟节点为S，它们的父节点为P，Sl和Sr为S节点的左右子节点为例，进行讲解，其中**N是父节点P的左子节点**，如果N是父节点P的右子节点，做对称处理。


3.1：N是新的根节点。这种情况下不用做任何处理，因为原先的节点也是一个根节点，相当于所有的路径都需要经过这个根节点，删除之后没有什么影响，而且新根也是黑色节点，符合所有特性，不需要进行调整

3.2: S节点是红色节点，那么P节点，Sl，Sr节点是黑色节点。在这种情况下，对P节点进行左选操作并且交换P和S的颜色。完成这2个操作之后，所有路径上的黑色节点没有变化，但是N节点有了一个黑色兄弟节点Sl和一个红色的父亲节点P，左子树删除节点后还有存在着少1个黑色节点路径的问题。接下来按照N节点新的位置(兄弟节点S是个黑色节点，父节点P是个红色节点)进行3.4、3.5或3.6情况处理	
![](http://7x2wh6.com1.z0.glb.clouddn.com/treemap10.jpg?imageView2/1/w/10000/h/10000/q/100|watermark/2/text/ZmFuZ2ppYW4wNDIzLmdpdGh1Yi5pbw==/font/5a6L5L2T/fontsize/500/fill/I0VGRUZFRg==/dissolve/100/gravity/SouthEast/dx/10/dy/10)

3.3：N的父亲节点P、兄弟节点S，还有S的两个子节点Sl，Sr均为黑色节点。在这种情况下，重绘S为红色。重绘之后路过S节点这边的路径跟N节点一样也少了一个黑色节点，但是出现了另外一个问题：不经过P节点的路径还是少了一个黑色节点。 接下来，要调整以P作为N递归调整树

![](http://7x2wh6.com1.z0.glb.clouddn.com/treemap11.jpg?imageView2/1/w/10000/h/10000/q/100|watermark/2/text/ZmFuZ2ppYW4wNDIzLmdpdGh1Yi5pbw==/font/5a6L5L2T/fontsize/500/fill/I0VGRUZFRg==/dissolve/100/gravity/SouthEast/dx/10/dy/10)

3.4：S和S的儿子节点Sl、Sr为黑色节点，但是N的父亲节点P为红色节点。在这种情况下，交换N的兄弟S与父亲P的颜色，颜色交换之后左子树多了1个黑色节点路径，刚好填补了左子树删除节点的少一个黑色节点路径的问题，而右子树的黑色路径没有改变，解决平衡问题

![](http://7x2wh6.com1.z0.glb.clouddn.com/treemap12.jpg?imageView2/1/w/10000/h/10000/q/100|watermark/2/text/ZmFuZ2ppYW4wNDIzLmdpdGh1Yi5pbw==/font/5a6L5L2T/fontsize/500/fill/I0VGRUZFRg==/dissolve/100/gravity/SouthEast/dx/10/dy/10)

3.5：S是黑色节点，S的左儿子节点Sl是红色，S的右儿子节点Sr是黑色节点。在这种情况下，在S上做右旋操作交换S和它新父亲的颜色。操作之后，左子树的黑色节点路径和右子树的黑色节点路径没有改变。但是现在N节点有了一个黑色的兄弟节点，黑色的兄弟节点有个红色的右儿子节点，满足了3.6的情况，按照3.6处理

![](http://7x2wh6.com1.z0.glb.clouddn.com/treemap13.jpg?imageView2/1/w/10000/h/10000/q/100|watermark/2/text/ZmFuZ2ppYW4wNDIzLmdpdGh1Yi5pbw==/font/5a6L5L2T/fontsize/500/fill/I0VGRUZFRg==/dissolve/100/gravity/SouthEast/dx/10/dy/10)

3.6：S是黑色节点，S的右儿子节点Sr为红色节点，S的左儿子Sl是黑色节点，P是红色或黑色节点。在这种情况下，N的父亲P做左旋操作，交换N父亲P和S的颜色，S的右子节点Sr变成黑色。这样操作以后，左子树的黑色路径+1，补了删除节点的黑色路径，右子树黑色路径不变，解决平衡问题

![](http://7x2wh6.com1.z0.glb.clouddn.com/treemap14.jpg?imageView2/1/w/10000/h/10000/q/100|watermark/2/text/ZmFuZ2ppYW4wNDIzLmdpdGh1Yi5pbw==/font/5a6L5L2T/fontsize/500/fill/I0VGRUZFRg==/dissolve/100/gravity/SouthEast/dx/10/dy/10)


了解了删除节点之后的平衡性调整之后，我们回过头来看本文一开始的例子进行节点删除的操作过程：


![](http://7x2wh6.com1.z0.glb.clouddn.com/treemap15.jpg?imageView2/1/w/10000/h/10000/q/100|watermark/2/text/ZmFuZ2ppYW4wNDIzLmdpdGh1Yi5pbw==/font/5a6L5L2T/fontsize/500/fill/I0VGRUZFRg==/dissolve/100/gravity/SouthEast/dx/10/dy/10)



TreeMap删除方法如下：

	private void deleteEntry(Entry<K,V> p) {
        modCount++;
        size--; // 节点个数 -1

        if (p.left != null && p.right != null) { // 如果要删除的节点有2个子节点，去找后继节点
            Entry<K,V> s = successor(p); // 找出后继节点
            p.key = s.key; // 后继节点的关键字赋值给删除节点
            p.value = s.value; // 后继节点的值赋值给删除节点
            p = s; // 改为删除后继节点
        }

        Entry<K,V> replacement = (p.left != null ? p.left : p.right); // 找出替代节点，左子树存在的话使用左子树，否则使用右子树。这个替代节点就是被删除节点的左子节点或右子节点

        if (replacement != null) { // 替代节点如果存在的话
            replacement.parent = p.parent; // 删除要删除的节点
            // 有子节点的删除节点先删除节点，然后再做平衡性调整
            if (p.parent == null) // 如果被删除节点的父节点为空，说明被删除节点是根节点
                root = replacement; // 用替代节点替代根节点
            else if (p == p.parent.left)
                p.parent.left  = replacement; // 用替代节点替代原先被删除的节点
            else
                p.parent.right = replacement; // 用替代节点替代原先被删除的节点

            p.left = p.right = p.parent = null;

            if (p.color == BLACK) // 被删除节点如果是黑色节点，需要进行平衡性调整
                fixAfterDeletion(replacement);
        } else if (p.parent == null) { // 如果被删除节点的父节点为空，说明被删除节点是根节点
            root = null; // 根节点的删除直接把根节点置空即可
        } else { //   如果要删除的节点没有子节点
            if (p.color == BLACK) // 如果要删除的节点是个黑色节点，需要进行平衡性调整
                fixAfterDeletion(p); // 调整平衡性，没有子节点的删除节点先进行平衡性调整

            if (p.parent != null) { // 没有子节点的删除节点平衡性调整完毕之后再进行节点删除
                if (p == p.parent.left)
                    p.parent.left = null;
                else if (p == p.parent.right)
                    p.parent.right = null;
                p.parent = null;
            }
        }
    }

	// 删除节点后的平衡性调整，对应之前分析的节点昵称，N、S、P、Sl、Sr
	private void fixAfterDeletion(Entry<K,V> x) {
        while (x != root && colorOf(x) == BLACK) { // N节点是黑色节点并且不是根节点就一直循环
            if (x == leftOf(parentOf(x))) { // 如果N是P的左子节点
                Entry<K,V> sib = rightOf(parentOf(x)); // sib就是N节点的兄弟节点S

                if (colorOf(sib) == RED) { // 如果S节点是红色节点，满足删除冲突3.2，对P节点进行左旋操作并交换P和S的颜色
                	// 交换P和S的颜色，S原先为红色，P原先为黑色(2个红色节点不能相连)
                    setColor(sib, BLACK); // S节点从红色变成黑色
                    setColor(parentOf(x), RED); // P节点从黑色变成红色
                    rotateLeft(parentOf(x)); // 删除冲突3.2中P节点进行左旋
                    sib = rightOf(parentOf(x)); // 左旋之后N节点有了一个黑色的兄弟节点和红色的父亲节点，S节点重新赋值成N节点现在的兄弟节点。接下来按照删除冲突3.4、3.5、3.6处理
                }
					
				// 执行到这里S节点一定是黑色节点，如果是红色节点，会按照冲突3.2交换成黑色节点
				// 如果S节点的左右子节点Sl、Sr均为黑色节点并且S节点也为黑色节点
                if (colorOf(leftOf(sib))  == BLACK &&
                    colorOf(rightOf(sib)) == BLACK) {
                    // 按照删除冲突3.3和3.4进行处理
                    // 如果是冲突3.3，说明P节点也是黑色节点
                    // 如果是冲突3.4，说明P节点是红色节点，P节点和S节点需要交换颜色
                    // 3.3和3.4冲突的处理结果S节点都为红色节点，但是3.4冲突处理完毕之后直接结束，而3.3冲突处理完毕之后继续调整
                    setColor(sib, RED); // S节点变成红色节点，如果是3.4冲突需要交换颜色，N节点的颜色交换在跳出循环进行
                    x = parentOf(x); // N节点重新赋值成N节点的父节点P之后继续递归处理
                } else { // S节点的2个子节点Sl，Sr中存在红色节点
                    if (colorOf(rightOf(sib)) == BLACK) { // 如果S节点的右子节点Sr为黑色节点，Sl为红色节点[Sl如果为黑色节点的话就在上一个if逻辑里处理了]，满足删除冲突3.5
                    	// 删除冲突3.5，对S节点做右旋操作，交换S和Sl的颜色，S变成红色节点，Sl变成黑色节点
                        setColor(leftOf(sib), BLACK); // Sl节点变成黑色节点
                        setColor(sib, RED); // S节点变成红色节点
                        rotateRight(sib); // S节点进行右旋操作
                        sib = rightOf(parentOf(x)); // S节点赋值现在N节点的兄弟节点
                    }
                    // 删除冲突3.5处理之后变成了删除冲突3.6或者一开始就是删除冲突3.6
                    // 删除冲突3.6，P节点做左旋操作，P节点和S接口交换颜色，Sr节点变成黑色
                    setColor(sib, colorOf(parentOf(x))); // S节点颜色变成P节点颜色，红色
                    setColor(parentOf(x), BLACK); // P节点变成S节点颜色，也就是黑色
                    setColor(rightOf(sib), BLACK); // Sr节点变成黑色
                    rotateLeft(parentOf(x)); // P节点做左旋操作
                    x = root; // 准备跳出循环
                }
            } else { // 如果N是P的右子节点，处理过程跟N是P的左子节点一样，左右对换即可
                Entry<K,V> sib = leftOf(parentOf(x));

                if (colorOf(sib) == RED) {
                    setColor(sib, BLACK);
                    setColor(parentOf(x), RED);
                    rotateRight(parentOf(x));
                    sib = leftOf(parentOf(x));
                }

                if (colorOf(rightOf(sib)) == BLACK &&
                    colorOf(leftOf(sib)) == BLACK) {
                    setColor(sib, RED);
                    x = parentOf(x);
                } else {
                    if (colorOf(leftOf(sib)) == BLACK) {
                        setColor(rightOf(sib), BLACK);
                        setColor(sib, RED);
                        rotateLeft(sib);
                        sib = leftOf(parentOf(x));
                    }
                    setColor(sib, colorOf(parentOf(x)));
                    setColor(parentOf(x), BLACK);
                    setColor(leftOf(sib), BLACK);
                    rotateRight(parentOf(x));
                    x = root;
                }
            }
        }

        setColor(x, BLACK); // 删除冲突3.4循环调出来之后N节点颜色设置为黑色 或者 删除节点只有1个红色子节点的时候，将顶上来的红色节点设置为黑色
    }

## 参考资料 ##


[http://dongxicheng.org/structure/red-black-tree/](http://dongxicheng.org/structure/red-black-tree/)

[http://blog.csdn.net/chenssy/article/details/26668941](http://blog.csdn.net/chenssy/article/details/26668941)

[http://zh.wikipedia.org/wiki/%E7%BA%A2%E9%BB%91%E6%A0%91](http://zh.wikipedia.org/wiki/%E7%BA%A2%E9%BB%91%E6%A0%91)

[http://www.cnblogs.com/fanzhidongyzby/p/3187912.html](http://www.cnblogs.com/fanzhidongyzby/p/3187912.html)