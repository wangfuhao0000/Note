# ArrayBlockingQueue解析

先看一下包含的成员变量：

一个Object数组用来存储元素，然后两个index用来标记取元素和存元素的下标，最经典的同步**使用了一个锁，两个条件变量来实现**。

```
//用来存储元素
final Object[] items;
//要取出元素的索引
int takeIndex;
//放入元素的索引
int putIndex;
//元素数量
int count;

/*
 * Concurrency control uses the classic two-condition algorithm
 * found in any textbook.
 */

/** Main lock guarding all access */
final ReentrantLock lock;

/** Condition for waiting takes */
private final Condition notEmpty;

/** Condition for waiting puts */
private final Condition notFull;
```

**1. 初始化**

初始化必须指定容量，**因为这是个环形队列所以容量要固定，不支持扩容**，默认为**非公平锁**。

```
public ArrayBlockingQueue(int capacity) {
    this(capacity, false);
}

public ArrayBlockingQueue(int capacity, boolean fair) {
    if (capacity <= 0)
        throw new IllegalArgumentException();
    this.items = new Object[capacity];  
    lock = new ReentrantLock(fair); //公平锁
    notEmpty = lock.newCondition();
    notFull =  lock.newCondition();
}
```

**2. 入队操作**

入队的时候要先上锁，加锁后添加元素，如果队列满了则返回false。然后此队列是一个环形队列，且加锁是加独占锁，保证成功的。

```
public boolean offer(E e) {
    checkNotNull(e);
    final ReentrantLock lock = this.lock;
    lock.lock();  //要先加锁
    try {
        if (count == items.length)  //队列满了
            return false;
        else {
            enqueue(e); //加到队列中
            return true;
        }
    } finally {
        lock.unlock();
    }
}

private void enqueue(E x) {
    final Object[] items = this.items;
    items[putIndex] = x;
    if (++putIndex == items.length)
        putIndex = 0;  //是个循环队列
    count++;
    notEmpty.signal();  //加入元素的时候通知队列不为null
}
```

这个方法和上面不同的是加锁是可中断的，且如果**队列满了是会阻塞**的。

```
public void put(E e) throws InterruptedException {
    checkNotNull(e);
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        while (count == items.length)
            notFull.await();  //队列满的时候，放在等待不满的队列中
        enqueue(e);
    } finally {
        lock.unlock();
    }
}
```

还有一个超时的方法，就是在指定时间内队列还是满的，则返回false。

```
public boolean offer(E e, long timeout, TimeUnit unit)
    throws InterruptedException {

    checkNotNull(e);
    long nanos = unit.toNanos(timeout);
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        while (count == items.length) {
            if (nanos <= 0)
                return false;  //固定时间内还不能假如，则return false
            nanos = notFull.awaitNanos(nanos);
        }
        enqueue(e);
        return true;
    } finally {
        lock.unlock();
    }
}
```

**3.出队操作**

首先上锁，如果队列为null则返回null，否则出队一个元素。

出队的操作步骤：

- 首先得到相应位置的元素，置相应位置为null
- 然后判断下标，因为是个循环队列，count--
- 然后更改遍历器
- 最后通知条件变量中的线程，队列不为空了

```
public E poll() {
    final ReentrantLock lock = this.lock;
    lock.lock();  //先上锁
    try {
        return (count == 0) ? null : dequeue(); //如果队列为null则直接返回
    } finally {
        lock.unlock();
    }
}

private E dequeue() {
    final Object[] items = this.items;
    @SuppressWarnings("unchecked")
    E x = (E) items[takeIndex];
    items[takeIndex] = null;  //出队后，置为null
    if (++takeIndex == items.length)  //循环队列
        takeIndex = 0;
    count--;
    if (itrs != null)  //这是个遍历器，用来保存
        itrs.elementDequeued();
    notFull.signal();  //出队列时，总是告诉队列中元素，队列不满了
    return x;
}
```

取走一个元素，是可以中断的，等待队列不为空的时候取走元素。**如果队列始终为空，则进入等待不为空的队列条件队列中**。

```
public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        while (count == 0)
            notEmpty.await();  //等队列不为null
        return dequeue();      //出队
    } finally {
        lock.unlock();
    }
}
```

带过期时间的出队，如果过期了返回null，否则就等队列不为null且等待时长为nanos。

```
public E poll(long timeout, TimeUnit unit) throws InterruptedException {
    long nanos = unit.toNanos(timeout);
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        while (count == 0) {
            if (nanos <= 0)
                return null;  //如果过期了，返回Null
            nanos = notEmpty.awaitNanos(nanos);
        }
        return dequeue();
    } finally {
        lock.unlock();
    }
}
```

**总结**

使用了一个数组来存储元素，两个下标index标记存元素和放元素时的位置，一个count用来存储元素的数量。

- ArrayBlockingQueue是一个环形的队列，所以在初始化时一定要指定队列的容量大小
- 使用了一个锁，两个条件变量。一个锁保证了入队和出队都要进行加锁操作，线程安全。两个条件变量保证等待取元素的线程和等待放元素的线程**可以进行阻塞操作，并被相应的唤醒**。
- 入队和出队都提供了**阻塞的**和**非阻塞的**操作，非阻塞操作在队列满了或者为空时能直接返回，阻塞操作在队列满了或者为空时能放利用条件变量到相应的阻塞队列中。每次入队和出队时都会去尝试唤醒两个阻塞队列里面的一个线程。

# LinkedBlockingQueue解析

先看看它的成员变量：

要注意的是这个和ArrayBlockingQueue不同的是使用了两个锁，取锁和拿锁默认的是**非公平锁**。这个区别是因为链表是一个**“单向”增长的**，即它只会在一头增长，在另一头减少。所以只需要**分别控制**添加元素和删除元素的原子性就可以了。同样也是因为这个原因，这里的size是一个AtomicInteger，这是因为**增加和减少可能同时执行**，所以保证size原子性更新。

```
// 容量，默认是Integet.MAX_VALUE
private final int capacity;

// 当前元素的数量
private final AtomicInteger count = new AtomicInteger();

// 头结点
transient Node<E> head;

// 尾结点
private transient Node<E> last;

// 独占锁，取元素(take、poll)时使用
private final ReentrantLock takeLock = new ReentrantLock();

// 取元素时的条件队列
private final Condition notEmpty = takeLock.newCondition();

// 独占锁，添加元素时(put、offer)时使用
private final ReentrantLock putLock = new ReentrantLock();

// 添加元素时的条件队列
private final Condition notFull = putLock.newCondition();
```

**1. 初始化**

```
public LinkedBlockingQueue() {
    this(Integer.MAX_VALUE);
}

//默认初始化头结点和尾结点指向一个空节点
public LinkedBlockingQueue(int capacity) {
    if (capacity <= 0) throw new IllegalArgumentException();
    this.capacity = capacity;
    last = head = new Node<E>(null); 
}
```

**2. 入队操作**

有两个函数put和offer。先说一下put方法，**put是阻塞的**，即如果队列满了，则会进入到放元素的阻塞队列中，等待队列不满。而**offer是非阻塞的**，如果队列满了直接返回false。有一个超时的入队操作，**如果某一段时间内未成功则返回null，否则一直尝试去入队**。

添加完元素后就去尝试唤醒那些等待队列不为空的线程，让他们能到队列里面去拿元素。

```
public void put(E e) throws InterruptedException {
    if (e == null) throw new NullPointerException();
    int c = -1;
    Node<E> node = new Node<E>(e);
    final ReentrantLock putLock = this.putLock;
    final AtomicInteger count = this.count;
    putLock.lockInterruptibly();
    try {
        while (count.get() == capacity) {
            notFull.await();
        }
        enqueue(node);
        c = count.getAndIncrement();
        if (c + 1 < capacity)
            notFull.signal();
    } finally {
        putLock.unlock();
    }
    if (c == 0)
        signalNotEmpty();
}

public boolean offer(E e) {
    if (e == null) throw new NullPointerException();
    final AtomicInteger count = this.count;
    if (count.get() == capacity)  //如果满了，直接返回false
        return false;
    int c = -1;
    Node<E> node = new Node<E>(e);
    final ReentrantLock putLock = this.putLock;
    putLock.lock();
    try {
        if (count.get() < capacity) {
            enqueue(node);
            c = count.getAndIncrement();
            if (c + 1 < capacity)
                notFull.signal();
        }
    } finally {
        putLock.unlock();
    }
    if (c == 0)
        signalNotEmpty();
    return c >= 0;
}

public boolean offer(E e, long timeout, TimeUnit unit)
    throws InterruptedException {

    if (e == null) throw new NullPointerException();
    long nanos = unit.toNanos(timeout);
    int c = -1;
    final ReentrantLock putLock = this.putLock;
    final AtomicInteger count = this.count;
    putLock.lockInterruptibly();
    try {
        while (count.get() == capacity) {
            if (nanos <= 0)
                return false;
            nanos = notFull.awaitNanos(nanos);
        }
        enqueue(new Node<E>(e));
        c = count.getAndIncrement();
        if (c + 1 < capacity)
            notFull.signal();
    } finally {
        putLock.unlock();
    }
    if (c == 0)
        signalNotEmpty();
    return true;
}

private void enqueue(Node<E> node) {
    last = last.next = node;   //入队列
}
```

**3. 出队操作**

同样的take操作是阻塞的，队列为空时会阻塞，等待队列不为空。而poll操作是非阻塞的，如果队列为空直接返回null。当然还有一个超时的poll，**即如果某一段时间内没有出队列则返回null，否则会一直尝试去出队**。

```
public E take() throws InterruptedException {
    E x;
    int c = -1;
    final AtomicInteger count = this.count;
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lockInterruptibly();
    try {
        while (count.get() == 0) {  //队列为空时，会阻塞
            notEmpty.await();
        }
        x = dequeue();
        c = count.getAndDecrement();
        if (c > 1)
            notEmpty.signal();
    } finally {
        takeLock.unlock();
    }
    if (c == capacity)
        signalNotFull();
    return x;
}

public E poll() {
    final AtomicInteger count = this.count;
    if (count.get() == 0)  //队列为空，直接返回null
        return null;
    E x = null;
    int c = -1;
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lock();
    try {
        if (count.get() > 0) {
            x = dequeue();
            c = count.getAndDecrement();
            if (c > 1)
                notEmpty.signal();
        }
    } finally {
        takeLock.unlock();
    }
    if (c == capacity)
        signalNotFull();
    return x;
}

public E poll(long timeout, TimeUnit unit) throws InterruptedException {
    E x = null;
    int c = -1;
    long nanos = unit.toNanos(timeout);
    final AtomicInteger count = this.count;
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lockInterruptibly();
    try {
        while (count.get() == 0) {
            if (nanos <= 0)
                return null;
            nanos = notEmpty.awaitNanos(nanos);
        }
        x = dequeue();
        c = count.getAndDecrement();
        if (c > 1)
            notEmpty.signal();
    } finally {
        takeLock.unlock();
    }
    if (c == capacity)
        signalNotFull();
    return x;
}

private E dequeue() {
    // assert takeLock.isHeldByCurrentThread();
    // assert head.item == null;
    Node<E> h = head;
    Node<E> first = h.next;
    h.next = h; // help GC
    head = first;
    E x = first.item;
    first.item = null;
    return x;
}
```

# PriorityBlockingQueue解析

先看一下包含的成员变量

一个数组用来存储堆内的元素，一个comparator用来指定排序规则，只有一个锁用来保证所有操作（入队、出队）的原子性，一个条件变量notEmpty用来存储因出队而阻塞的线程。

```
//默认长度11
private static final int DEFAULT_INITIAL_CAPACITY = 11;

//最大长度，因为数组对象的对象头，可能会溢出
private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

 //默认是个小顶堆
private transient Object[] queue;

//元素数量
private transient int size;

//如果未指定，使用元素的自然排序
private transient Comparator<? super E> comparator;

//锁
private final ReentrantLock lock;

//条件变量，表示队列不为null
private final Condition notEmpty;

//自旋锁，通过CAS实现
private transient volatile int allocationSpinLock;

//用来序列化
private PriorityQueue<E> q;
```

**1. 初始化**

初始化默认大小为11。

```
public PriorityBlockingQueue() {  
    this(DEFAULT_INITIAL_CAPACITY, null);
}
public PriorityBlockingQueue(int initialCapacity) {
    this(initialCapacity, null);
}
public PriorityBlockingQueue(int initialCapacity,
                             Comparator<? super E> comparator) {
    if (initialCapacity < 1)  
        throw new IllegalArgumentException();
    this.lock = new ReentrantLock();
    this.notEmpty = lock.newCondition();
    this.comparator = comparator;
    this.queue = new Object[initialCapacity];
}
```

**2. 入队**

与前面不同的是，此处入队的函数不管是add还是put最后调用的都是offer，所以只需看offer即可。

- 首先去尝试加锁，**如果拿不到锁则会被阻塞**。所以此方法**总会返回true**。
- 然后进行容量的判断，如果不够了则进行扩容。**int** newCap = oldCap + ((oldCap < 64) ? (oldCap + 2) : (oldCap >> 1)); 如果原来容量小于64，则是二倍原来容量加2，否则是原来的1.5倍容量。
- 然后将元素入队，如果有自定义的Comparator，则使用自己的，否则是默认的。
- 最后通知要出队的线程，线程不为空了！！

```
public boolean offer(E e) {
    if (e == null)
        throw new NullPointerException();
    final ReentrantLock lock = this.lock;
    lock.lock();  //添加元素前先加锁，如果没抢到就会阻塞
    int n, cap;
    Object[] array;
    while ((n = size) >= (cap = (array = queue).length))
        tryGrow(array, cap);
    try {
        Comparator<? super E> cmp = comparator;
        if (cmp == null)
            siftUpComparable(n, e, array);
        else
            siftUpUsingComparator(n, e, array, cmp);
        size = n + 1;
        notEmpty.signal();  //添加元素，通知其他线程队列不为空，可以出队
    } finally {
        lock.unlock();
    }
    return true;
}
```

**3. 出队**

出队分为两个poll和take。poll尝试加锁，然后出队列。当然有一个额外的超时poll，即在指定时间内如果未能出队则返回null。而take则是会一直自旋的去尝试出队，**如果不行就会放到上面的条件队列中**，等待队列不为空。**所以一个是被阻塞在了加锁的过程中，而一个是阻塞在了条件队列中**。

```
public E poll() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        return dequeue();
    } finally {
        lock.unlock();
    }
}

public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    E result;
    try {
        while ( (result = dequeue()) == null)
            notEmpty.await();
    } finally {
        lock.unlock();
    }
    return result;
}

public E poll(long timeout, TimeUnit unit) throws InterruptedException {
    long nanos = unit.toNanos(timeout);
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    E result;
    try {
        while ( (result = dequeue()) == null && nanos > 0)
            nanos = notEmpty.awaitNanos(nanos);
    } finally {
        lock.unlock();
    }
    return result;
}
```

# ConcurrentLinkedQueue原理探究

ConcurrentLinkedQueue是**线程安全**的**无界、非阻塞**队列，其底层数据结构使用**单向链表**实现，对于入队和出队操作使用**CAS来实现线程安全**。

![img](https://note.youdao.com/yws/public/resource/56347de0c9d0b624bcbb02366ca7a2b8/xmlnote/44D322E5968E4A988582BF4E58A1170E/88F3B07260144B7AAC38EE19CB732479/11620)

它的内部有两个volatile类型的Node节点head和tail分别用来存放队列的首、尾节点，默认两个节点都是指向item为null的哨兵节点。

```
public ConcurrentLinkedQueue() {
    head = tail = new Node<E>(null);
}
```

在Node节点的内部维护一个使用volatile修饰的变量**item**，用来存放节点的值；**next**用来存放链表的下一个节点，从而链接为一个单向无界链表。其内部则使用**UNSafe工具类**提供CAS算法保证出入队时操作链表的原子性。

**1. offer操作**

```
public boolean offer(E e) {
    checkNotNull(e); //若为null则抛出异常
    //构造节点，内部使用unsafe保证构造的原子性
    final Node<E> newNode = new Node<E>(e); 

    for (Node<E> t = tail, p = t;;) { //不断获取tail
        Node<E> q = p.next;
        if (q == null) { // p是最后一个节点，则执行插入
            if (p.casNext(null, newNode)) {
                // 原子设置p的next为创建的新节点
                // for e to become an element of this queue,
                // and for newNode to become "live".
                if (p != t) // hop two nodes at a time
                    casTail(t, newNode);  // Failure is OK.
                return true;
            }
            // Lost CAS race to another thread; re-read next
        }
        else if (p == q)
            // We have fallen off list.  If tail is unchanged, it
            // will also be off-list, in which case we need to
            // jump to head, from which all live nodes are always
            // reachable.  Else the new tail is a better bet.
            p = (t != (t = tail)) ? t : head;
        else
            // Check for tail updates after two hops.
            p = (p != t && t != (t = tail)) ? t : q;
    }
}
```

# SynchronousQueue解析

