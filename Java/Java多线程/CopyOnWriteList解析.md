CopyOnWriteArrayList是一个线程安全的ArrayList，对其进行的修改操作都是**在底层的一个复制的数组（快照）上进行的**，也就是使用了**写时复制策略**。如图所示是CopyOnWriteArrayList的类图结构：
![CopyOnWriteArrayList类图](https://img-blog.csdnimg.cn/2019061323355451.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3JlYWNod2FuZw==,size_16,color_FFFFFF,t_70)
能够看到，每个CopyOnWriteArrayList对象都有一个**`volatile array`**数组用来存放具体元素，**而ReenTrantLock则用来保证只有一个线程对Array进行修改**。ReenTrantLock本身是一个独占锁，同时只有一个线程能够获取。接下来看一下其中的一些方法代码。

### 初始化

共有三个构造函数：

```java
public CopyOnWriteArrayList() {
    setArray(new Object[0]);			//创建一个大小为0的Object数组作为array初始值
}
public CopyOnWriteArrayList(E[] toCopyIn) {
		//创建一个list，其内部元素是toCopyIn的的副本
    setArray(Arrays.copyOf(toCopyIn, toCopyIn.length, Object[].class));
}
//将传入参数集合中的元素复制到本list中
public CopyOnWriteArrayList(Collection<? extends E> c) {
    Object[] elements;
    if (c.getClass() == CopyOnWriteArrayList.class)
        elements = ((CopyOnWriteArrayList<?>)c).getArray();
    else {
        elements = c.toArray();
        // c.toArray might (incorrectly) not return Object[] (see 6260652)
        if (elements.getClass() != Object[].class)
            elements = Arrays.copyOf(elements, elements.length, Object[].class);
    }
    setArray(elements);
}
```

setArray方法很简单：

```java
final void setArray(Object[] a) {
    array = a;
}
```

### 添加元素

添加元素有很多方法，包括add(E e), add(int index, E element)等，原理基本上相同，所以我们只看add(E e)的源码。

```java
public boolean add(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();		//先加锁
    try {
        Object[] elements = getArray();
        int len = elements.length;
        Object[] newElements = Arrays.copyOf(elements, len + 1);		//复制到新数组中
        newElements[len] = e;		//在新数组中添加元素
        setArray(newElements);		//将元素设置为新数组
        return true;
    } finally {
        lock.unlock();
    }
}
```

流程如下：

- 要想对其进行修改，需要先加锁
- 将原来的数组内容复制到一个新的数组中，并在新数组中进行修改
- 将自己的引用指向新的修改过的数组
- 释放锁

### 获取指定位置元素

使用E get(int index)方法获取下标为index的元素：

```java
	public E get(int index) {
        return get(getArray(), index);
    }
	
	final Object[] getArray() {
        return array;
    }

	private E get(Object[] a, int index) {
        return (E) a[index];
    }
1234567891011
```

这个方法是线程不安全的，因为这个分成了两步，分别是获取数组和获取元素，而且**中间过程没有加锁**。假设当前线程在获取数组（执行getArray()）后，其他线程修改了这个CopyOnWriteArrayList，那么它里面的元素就会改变，但此时当前线程返回的仍然是旧的数组，所以返回的元素就不是最新的了，这就是**写时复制策略产生的弱一致性问题**。

### 修改指定元素

使用`E set (int index, E element)`修改list中指定元素的值，代码如下：

```java
public E set(int index, E element) {
    final ReentrantLock lock = this.lock;
    lock.lock();		//加锁
    try {
        Object[] elements = getArray();
        E oldValue = get(elements, index);		//先得到要修改的旧值

        if (oldValue != element) {				//值确实修改了
            int len = elements.length;
            //将array复制到新数组，并进行修改，并设置array为新数组
            Object[] newElements = Arrays.copyOf(elements, len);			
            newElements[index] = element;
            setArray(newElements);
        } else {
            // 虽然值确实没改，但要保证volatile语义，需重新设置array
            setArray(elements);
        }
        return oldValue;
    } finally {
        lock.unlock();
    }
}
```

大概流程如下：

- 首先加锁
- 然后得到原来的值，和要修改的值进行比较：
  - 如果确实修改了，则复制一个新数组，并在新数组中更改值
  - 没有修改，则也要执行`setArray()`来设置一下，保证volatile语义
- 释放锁

### 删除元素

使用`public E remove(int index)`方法，代码如下：

```java
public E remove(int index) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        E oldValue = get(elements, index);			//得到要删除的元素
        int numMoved = len - index - 1;
        if (numMoved == 0)
            setArray(Arrays.copyOf(elements, len - 1));
        else {
            Object[] newElements = new Object[len - 1];
            System.arraycopy(elements, 0, newElements, 0, index);
            System.arraycopy(elements, index + 1, newElements, index,
                             numMoved);
            setArray(newElements);
        }
        return oldValue;
    } finally {
        lock.unlock();
    }
}
```

大概流程如下：

- 先加锁
- 得到要删除的元素下标
  - 如果是最后一个元素，则直接设置数组为**原数组除去最后一个元素的拷贝**
  - 否则分两次拷贝到新数组
- 解锁，返回删除的元素

### 弱一致性的迭代器

我们先看一下迭代器是怎么使用的：

```java
public static void main(String[] args) {
    CopyOnWriteArrayList<String> arrayList = new CopyOnWriteArrayList<>();
    arrayList.add("hello");
    arrayList.add("alibaba");

    Iterator<String> itr = arrayList.iterator();
    while (((Iterator) itr).hasNext())
        System.out.println(itr.next());
}
```

很简单，那弱一致性是怎么回事呢，它是指**返回迭代器后，其他线程对list的增删改对迭代器是不可见的**。接下来看一下为什么会这样：

```java
public Iterator<E> iterator() {
    return new COWIterator<E>(getArray(), 0);	//返回一个COWIterator对象
}

static final class COWIterator<E> implements ListIterator<E> {
    /** 数组array快照 */
    private final Object[] snapshot;
    /** 数组下标  */
    private int cursor;

    private COWIterator(Object[] elements, int initialCursor) {
        cursor = initialCursor;
        snapshot = elements;  // 指向了原来的数组，当修改后，它仍然是不变的
    }

    public boolean hasNext() {
        return cursor < snapshot.length;
    }

    @SuppressWarnings("unchecked")
    public E next() {
        if (! hasNext())
            throw new NoSuchElementException();
        return (E) snapshot[cursor++];
		}
}
```

在调用`iterator()`方法后，会返回一个COWIterator对象，COWIterator对象的snapshot变量保存了当前list的内容，cursor是遍历list时数据的下标。

那么为什么说snapshot是list的快照呢，明明传的是引用。其实这就和CopyOnWriteArrayList本身有关了，如果在返回迭代器后没有对里面的数组array进行修改，则这两个变量指向的确实是同一个数组；**但是若修改了，则根据前面所讲，它是会新建一个数组，然后将修改后的数组复制到新建的数组，而老的数组就会被“丢弃”**，所以如果修改了数组，则此时snapshot指向的还是原来的数组，而array变量已经指向了新的修改后的数组了。这也就说明获取迭代器后，使用迭代器元素时，其他线程对该list的增删改不可见，因为他们操作的是两个不同的数组，这就是弱一致性。

接下来就演示一下这个现象：

```java
public class copylist {

    private static volatile CopyOnWriteArrayList<String> arrayList = new CopyOnWriteArrayList<>();

    public static void main(String[] args) throws InterruptedException{
        arrayList.add("hello");
        arrayList.add("alibaba");
        arrayList.add("welcome");
        arrayList.add("to");
        arrayList.add("hangzhou");

        Thread threadOne = new Thread(new Runnable() {
            @Override
            public void run() {
                arrayList.set(1, "baba");
                arrayList.remove(2);
                arrayList.remove(3);
            }
        });

        Iterator<String> itr = arrayList.iterator();

        threadOne.start();
        threadOne.join();

        while (itr.hasNext())
            System.out.println(itr.next());
    }
}
```

运行结果如下，说明虽然线程threadOne改变了这个list，但是获取了迭代器后，它指向的还是旧的数组，所以遍历的时候还是旧的数组内容。所以==获取迭代器的操作必须在子线程操作之前进行。

```java
hello
alibaba
welcome
to
hangzhou
12345
```

### 总结

CopyOnWriteArrayList使用**写时复制**策略保证list的一致性，而**获取–修改–写入三个步骤不是原子性，所以需要一个独占锁保证修改数据时只有一个线程能够进行**。另外，CopyOnWriteArrayList提供了弱一致性的迭代器，从而保证在获取迭代器后，其他线程对list的修改是不可见的，**迭代器遍历的数组是一个快照**。