## ArrayList解析

ArrayList就是数组列表，主要用来装载数据，当我们装载的是基本类型的数据int，long，boolean，short，byte…的时候我们只能存储他们对应的包装类，它的主要底层实现是数组`Object[] elementData`。和LinkedList相比，它的查找和访问元素的速度较快，但新增，删除的速度较慢。

### 为啥线程不安全还使用他呢？

因为我们正常使用的场景中，都是用来查询，不会涉及太频繁的增删，如果涉及频繁的增删，可以使用LinkedList，如果你需要线程安全就使用Vector，这就是三者的区别了，实际开发过程中还是ArrayList使用最多的。

您说它的底层实现是数组，但是数组的大小是定长的，如果我们不断的往里面添加数据的话，不会有问题吗？

ArrayList可以通过构造方法在初始化的时候指定底层数组的大小。

通过无参构造方法的方式ArrayList()初始化，则赋值底层数Object[] elementData为一个默认空数组Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {}所以数组容量为0，**只有真正对数据进行添加add时，才分配默认DEFAULT_CAPACITY = 10的初始容量**。

大家可以分别看下他的无参构造器和有参构造器，无参就是默认大小，有参会判断参数。

```java
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

/**
 * Constructs an empty list with an initial capacity of ten.
 */
public ArrayList() {
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}
```

数组的长度是有限制的，而ArrayList是可以存放任意数量对象，长度不受限制，它就是通过**数组扩容**的方式去实现的。**扩容后的大小应该是`int newCapacity = oldCapacity + (oldCapacity >> 1);`，即1.5倍的原来容量。**

因为我们在使用ArrayList的时候一般不会设置初始值的大小，**那ArrayList默认的大小就刚好是10**。

```java
/**
 * Default initial capacity.
 */
private static final int DEFAULT_CAPACITY = 10;
```

### 扩容

他增删很慢，你能说一下ArrayList在增删的时候是怎么做的么？主要说一下他为啥慢。我分别说一下他的新增的逻辑吧。

他有指定index新增，也有直接新增的，在这之前他会有一步校验长度的判断`ensureCapacityInternal`，就是说如果长度不够，是需要扩容的。

```java
public boolean add(E e) {
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    elementData[size++] = e;
    return true;
}
```



在扩容的时候，老版本的jdk和8以后的版本是有区别的，8之后的效率更高了，采用了位运算，**右移**一位，其实就是除以2这个操作。

![img](https://note.youdao.com/yws/public/resource/7a73a1639de2e76d59502a80ee58fce7/xmlnote/WEBa89231883bfc9d7ca55fcc4bab3eb908/46B48A21A73A41A2B936E56C5FBB63AC/17670)

指定位置新增的时候，在校验之后的操作很简单，就是**数组的copy**，大家可以看下代码。

![img](https://note.youdao.com/yws/public/resource/7a73a1639de2e76d59502a80ee58fce7/xmlnote/WEBa89231883bfc9d7ca55fcc4bab3eb908/32425441BD214B05BFE84896179B019B/17674)

不知道大家看懂**arraycopy**的代码没有，我画个图解释下，你可能就明白一点：

比如有下面这样一个数组我需要在index 5的位置去新增一个元素A

![img](https://note.youdao.com/yws/public/resource/7a73a1639de2e76d59502a80ee58fce7/xmlnote/WEBa89231883bfc9d7ca55fcc4bab3eb908/A70996DC98A3436ABF7B0D6833954BCE/17675)

那从代码里面我们可以看到，他复制了一个数组，是从index 5的位置开始的，然后把它放在了index 5+1的位置

![img](https://note.youdao.com/yws/public/resource/7a73a1639de2e76d59502a80ee58fce7/xmlnote/WEBa89231883bfc9d7ca55fcc4bab3eb908/94E4D92815B8436C9FDEDA46E4EA4A2A/17658)

给我们要新增的元素腾出了位置，然后在index的位置放入元素A就完成了新增的操作了

![img](https://note.youdao.com/yws/public/resource/7a73a1639de2e76d59502a80ee58fce7/xmlnote/WEBa89231883bfc9d7ca55fcc4bab3eb908/CE7B48616AE746BF89254C0A718DBE7F/17659)

至于为啥说他效率低，我想我不说你也应该知道了，我这只是在一个这么小的List里面操作，要是我去一个几百几千几万大小的List新增一个元素，那就需要后面所有的元素都复制，然后如果再涉及到扩容啥的就更慢了不是嘛。

我问你个真实的场景，这个问题很少人知道，你可要好好回答哟！

### ArrayList（int initialCapacity）会不会初始化数组大小？

**不会初始化数组大小！**

而且将构造函数与initialCapacity结合使用，然后使用set（）会抛出异常，尽管该数组已创建，但是大小设置不正确。

使用sureCapacity（）也不起作用，因为它基于elementData数组而不是大小。

还有其他副作用，这是因为带有sureCapacity（）的静态DEFAULT_CAPACITY。

进行此工作的唯一方法是在使用构造函数后，根据需要使用add（）多次。

大家可能有点懵，我直接操作一下代码，大家会发现我们虽然对ArrayList设置了初始大小，但是我们打印List大小的时候还是0，我们操作下标set值的时候也会报错，数组下标越界。

因为这里设置元素会有一个rangeCheck，**它检测的不是容量和索引的大小，而是size和索引的大小**。

```
private void rangeCheck(int index) {
    if (index >= size)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}
```

再结合源码，大家仔细品读一下，这是Java Bug里面的一个经典问题了，还是很有意思的，大家平时可能也不会注意这个点。

![img](https://note.youdao.com/yws/public/resource/7a73a1639de2e76d59502a80ee58fce7/xmlnote/WEBa89231883bfc9d7ca55fcc4bab3eb908/A7BDE84377244F7FA50C449408458D56/17665)

ArrayList插入删除一定慢么？

取决于你删除的元素离数组末端有多远，ArrayList拿来作为堆栈来用还是挺合适的，push和pop操作完全不涉及数据移动操作。

那他的删除怎么实现的呢？

删除其实跟新增是一样的，不过叫是叫删除，但是在代码里面我们发现，他还是在copy一个数组。

为啥是copy数组呢？

![img](https://note.youdao.com/yws/public/resource/7a73a1639de2e76d59502a80ee58fce7/xmlnote/WEBa89231883bfc9d7ca55fcc4bab3eb908/3763550BCEBB456BAF80D303C73BD6AB/17669)

继续打个比方，我们现在要删除下面这个数组中的index5这个位置

![img](https://note.youdao.com/yws/public/resource/7a73a1639de2e76d59502a80ee58fce7/xmlnote/WEBa89231883bfc9d7ca55fcc4bab3eb908/86A25B57AA2B43B8A5A34C1FB921867C/17662)

那代码他就复制一个index5+1开始到最后的数组，然后把它放到index开始的位置

![img](https://note.youdao.com/yws/public/resource/7a73a1639de2e76d59502a80ee58fce7/xmlnote/WEBa89231883bfc9d7ca55fcc4bab3eb908/763C480250E141F49EAA84CC18064459/17664)

index5的位置就成功被”删除“了其实就是被覆盖了，给了你被删除的感觉。

同理他的效率也低，因为数组如果很大的话，一样需要复制和移动的位置就大了。

ArrayList是线程安全的么？

当然不是，线程安全版本的数组容器是Vector。

Vector的实现很简单，就是把所有的方法统统加上synchronized就完事了。

你也可以不使用Vector，用Collections.synchronizedList把一个普通ArrayList包装成一个线程安全版本的数组容器也可以，原理同Vector是一样的，就是给所有的方法套上一层synchronized。

ArrayList用来做队列合适么？

队列一般是FIFO（先入先出）的，如果用ArrayList做队列，就需要在数组尾部追加数据，数组头部删除数组，反过来也可以。

但是无论如何总会有一个操作会涉及到数组的数据搬迁，这个是比较耗费性能的。

**结论**：ArrayList不适合做队列。

那数组适合用来做队列么？数组是非常合适的。

比如ArrayBlockingQueue内部实现就是一个环形队列，它是一个定长队列，内部是用一个定长数组来实现的。

另外著名的Disruptor开源Library也是用环形数组来实现的超高性能队列，具体原理不做解释，比较复杂。

简单点说就是使用两个偏移量来标记数组的读位置和写位置，如果超过长度就折回到数组开头，前提是它们是**定长数组**。

ArrayList的遍历和LinkedList遍历性能比较如何？

论遍历ArrayList要比LinkedList快得多，ArrayList遍历最大的优势在于内存的连续性，CPU的内部缓存结构会缓存连续的内存片段，可以大幅降低读取内存的性能开销。

能跟我聊一下LinkedList相关的东西么？

**总结**

ArrayList就是动态数组，用MSDN中的说法，就是Array的复杂版本，它提供了动态的增加和减少元素，实现了`Collection`和`List`接口，灵活的设置数组的大小等好处。

**ArrayList常用的方法总结**

- boolean add(E e)

将指定的元素添加到此列表的尾部。

- void add(int index, E element)

将指定的元素插入此列表中的指定位置。

- boolean addAll(Collection c)

按照指定 collection 的迭代器所返回的元素顺序，将该 collection 中的所有元素添加到此列表的尾部。

- boolean addAll(int index, Collection c)

从指定的位置开始，将指定 collection 中的所有元素插入到此列表中。

- void clear()

移除此列表中的所有元素。

- Object clone()

返回此 ArrayList 实例的浅表副本。

- boolean contains(Object o)

如果此列表中包含指定的元素，则返回 true。

- void ensureCapacity(int minCapacity)

如有必要，增加此 ArrayList 实例的容量，以确保它至少能够容纳最小容量参数所指定的元素数。

- E get(int index)

返回此列表中指定位置上的元素。

- int indexOf(Object o)

返回此列表中首次出现的指定元素的索引，或如果此列表不包含元素，则返回 -1。

- boolean isEmpty()

如果此列表中没有元素，则返回 true

- int lastIndexOf(Object o)

返回此列表中最后一次出现的指定元素的索引，或如果此列表不包含索引，则返回 -1。

- E remove(int index)

移除此列表中指定位置上的元素。

- boolean remove(Object o)

移除此列表中首次出现的指定元素（如果存在）。

- protected void removeRange(int fromIndex, int toIndex)

移除列表中索引在 fromIndex（包括）和 toIndex（不包括）之间的所有元素。

- E set(int index, E element)

用指定的元素替代此列表中指定位置上的元素。

- int size()

返回此列表中的元素数。

- Object[] toArray()

按适当顺序（从第一个到最后一个元素）返回包含此列表中所有元素的数组。

- T[] toArray(T[] a)

按适当顺序（从第一个到最后一个元素）返回包含此列表中所有元素的数组；返回数组的运行时类型是指定数组的运行时类型。

- void trimToSize()

## LinkedList解析

**1. 数据结构**

就三个属性：size、头结点、尾节点。而链表是一个**双向链表**

```java
transient int size = 0;
transient Node<E> first;
transient Node<E> last;

private static class Node<E> {
    E item;
    Node<E> next;  //是一个双向链表
    Node<E> prev;

    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }
}
```

**2. 初始化**

初始化基本上就是什么也不做

```
public LinkedList() {
}
```

**3. 添加元素**

先看两个方法，头插和尾插，要注意到初始化的时候**头结点和尾节点其实都是null的，即不会初始化**。所以在加入节点的时候就要判断链表是否为null，为null则会让头结点和尾节点都指向此节点。

```java
private void linkFirst(E e) {
    final Node<E> f = first;
    final Node<E> newNode = new Node<>(null, e, f);
    first = newNode;
    if (f == null)  //说明链表为null，则新加入的结点也是尾节点
        last = newNode;
    else
        f.prev = newNode;
    size++;
    modCount++;
}

/**
 * Links e as last element.
 */
void linkLast(E e) {
    final Node<E> l = last;
    final Node<E> newNode = new Node<>(l, e, null);
    last = newNode;  //说明链表为null，则新加入的结点也是头结点
    if (l == null)
        first = newNode;
    else
        l.next = newNode;
    size++;
    modCount++;
}
```

**执行add时其实就是尾插**。

**4. 删除元素**

先看一个方法unlink，它就是把一个结点从链表中拿下来，并返回删除的元素。

而删除元素就是遍历链表，找到指定的元素然后将它从链表中拿下来。当然遍历过程中也分元素为null何不为null的情况。

```java
E unlink(Node<E> x) {
    // assert x != null;
    final E element = x.item;
    final Node<E> next = x.next;  //存储下它的前驱和后继
    final Node<E> prev = x.prev;

    if (prev == null) {  //如果是头结点，则新的头结点就是后继
        first = next;
    } else {
        prev.next = next;
        x.prev = null;
    }

    if (next == null) {  //如果也是尾节点
        last = prev;
    } else {
        next.prev = prev;
        x.next = null;
    }

    x.item = null;  //帮助GC
    size--;
    modCount++;
    return element;
}

public boolean remove(Object o) {
    if (o == null) {
        for (Node<E> x = first; x != null; x = x.next) {
            if (x.item == null) {
                unlink(x);
                return true;
            }
        }
    } else {
        for (Node<E> x = first; x != null; x = x.next) {
            if (o.equals(x.item)) {
                unlink(x);
                return true;
            }
        }
    }
    return false;
}
```

**5. 扩容**

链表是个无限长的，不需要扩容。

要注意LinkedList实现了**Deque接口**，所以是可以作为一个队列来使用的。

## Vector、Stack解析

- Vector就是在方法上加上了synchronized，但要注意这只能保证单个方法的线程安全，**对于复合操作（调用两个方法）就可能会不安全**，所以可在复合操作前加锁。

- Stack类就是继承的Vector，所以它也是线程安全的。它就是不停的在**数组的末尾**进行添加和修改。

