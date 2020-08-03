### 初识 ReentrantLock

ReentrantLock 位于 java.util.concurrent.locks 包下，它实现了 Lock 接口和 Serializable 接口。

![img](https://assets.javadoop.com/imgs/20511488/default/ReetrantLock1.jpg)

`ReentrantLock` 是一把可重入锁和互斥锁，它具有与` synchronized `关键字相同的含有隐式监视器锁（monitor）的基本行为和语义，但是它比 synchronized 具有更多的方法和功能。

### ReentrantLock 基本方法

#### 构造方法

ReentrantLock 类中带有两个构造函数，一个是默认的构造函数，不带任何参数；一个是带有 `fair` 参数的构造函数，此参数指定创建的锁是否是公平的，默认的情况下创建的是**非公平锁**。

```java
public ReentrantLock() {  
  	sync = new NonfairSync(); 
} 
public ReentrantLock(boolean fair) {  
  sync = fair ? new FairSync() : new NonfairSync(); 
}
```

`FairSync` 和 `NonfairSync` 都是 `ReentrantLock` 的内部类，继承于 `Sync` 类，下面来看一下它们的继承结构，便于梳理。

![img](https://assets.javadoop.com/imgs/20511488/default/ReetrantLock2.jpg)

```java
abstract static class Sync extends AbstractQueuedSynchronizer {...}
static final class FairSync extends Sync {...} 
static final class NonfairSync extends Sync {...}
```

在多线程尝试加锁时，如果是公平锁，那么**锁获取的机会是相同的**。否则，如果是非公平锁，那么 ReentrantLock 则**不会保证每个锁的访问顺序**。

#### 公平锁的加锁（lock）流程详解

通常情况下，使用多线程访问公平锁的效率会非常低（通常情况下会慢很多），但是 ReentrantLock 会保证每个线程都会公平的持有锁，线程饥饿的次数比较小。锁的公平性并不能保证线程调度的公平性。可以从源码的角度了解如何 ReentrantLock 是如何实现这两种锁的。

![img](https://assets.javadoop.com/imgs/20511488/default/ReetrantLock3.jpg)

如上图所示，公平锁的加锁流程要比非公平锁的加锁流程简单，下面要聊一下具体的流程了，请小伙伴们备好板凳。

下面先看一张流程图，这张图是 acquire 方法的三条主要流程

![img](https://assets.javadoop.com/imgs/20511488/default/ReetrantLock4.jpg)

##### 第一条路线：tryAcquire

顾名思义尝试获取，也就是说可以成功获取锁，也可以获取锁失败。调用的是 ReentrantLock 的 tryAcquire 方法；

```java
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
      	// 首先判断前面没有等待的线程，
      	// 然后才尝试改变状态量并设置自己为独占线程
        if (!hasQueuedPredecessors() &&   
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
  	// 自己当前正在持有锁，则增加状态量
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```



首先会取得当前线程，然后去读取当前锁的同步状态，还记得锁的四种状态吗？分别是 无锁、偏向锁、轻量级锁和重量级锁，如果你不是很明白的话，请参考博主这篇文章（[**不懂什么是锁？看看这篇你就明白了**](https://mp.weixin.qq.com/s?__biz=MzU2NDg0OTgyMA==&mid=2247484919&idx=1&sn=b36b9b84ad50559210b6f21901a882b9&chksm=fc45f804cb327112bb4301191d8f3a464244ca7cb93da73f3b1d8ff28821d6ac32e2512d66e0&token=1671614453&lang=zh_CN&scene=21#wechat_redirect)），如果判断同步状态是 0 的话，就证明是无锁的，参考下面这幅图( 1bit 表示的是是否偏向锁 )

![img](https://assets.javadoop.com/imgs/20511488/default/ReetrantLock5.jpg)

- 如果是无锁（也就是没有加锁），说明是第一次上锁，首先会先判断一下队列中是否有比当前线程等待时间更长的线程（hasQueuedPredecessors）；然后通过 CAS 方法原子性的更新锁的状态，CAS 通过 C 底层机制保证原子性，这个你不需要考虑它。如果既没有排队的线程而且使用 CAS 方法成功的把 0 -> 1 （偏向锁），那么当前线程就会获得**偏向锁**，记录获取锁的线程为当前线程。

- else if 逻辑，如果读取的同步状态是1，说明已经线程获取到了锁，那么就先判断当前线程是不是获取锁的线程，如果是的话，记录一下获取锁的次数 + 1，也就是说，只有同步状态为 0 的时候是无锁状态。如果当前线程不是获取锁的线程，直接返回 false。

acquire 方法会先查看同步状态是否获取成功，如果成功则方法结束返回，也就是 !tryAcquire == false ，若失败则先调用 addWaiter 方法再调用 acquireQueued 方法

##### 第二条路线 addWaiter

```java
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    // 先快速尝试在队尾加入节点
  	// 如果没有加入成功，则调用enq方法通过循环（CAS）加入
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    enq(node);
    return node;
}
```

这里首先把当前线程和 Node 的节点类型进行封装，Node 节点的类型有两种，**EXCLUSIVE **和**SHARED** ，前者为独占模式，后者为共享模式。

- 首先会进行 tail 节点的判断，有没有尾节点，其实没有头节点也就相当于没有尾节点，如果有尾节点，就会原子性的将当前节点插入同步队列中，再执行 enq 入队操作，入队操作相当于原子性的把节点插入队列中。
- 如果当前同步队列尾节点为null，说明当前线程是第一个加入同步队列进行等待的线程。

##### 第三条路线 acquireQueued

```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
          	// 假如先驱节点是头结点，且能获取到独占锁
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

会进行无限循环中，然后主要有两个分支判断

- 循环中每次都会判断给定当前节点的先驱节点，如果没有先驱节点会直接抛出空指针异常，直到返回 true。
- 然后判断给定节点的先驱节点是不是头节点，并且当前节点能否获取独占式锁，
  - **如果是头节点并且成功获取独占锁后**，队列头指针指向当前节点，然后释放前驱节点。
  - 如果没有获取到独占锁，就会进入 shouldParkAfterFailedAcquire 和 parkAndCheckInterrupt 方法中，我们贴出这两个方法的源码

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL)
        /*
         * This node has already set status asking a release
         * to signal it, so it can safely park.
         */
        return true;
    if (ws > 0) {
        /*
         * Predecessor was cancelled. Skip over predecessors and
         * indicate retry.
         */
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        /*
         * waitStatus must be 0 or PROPAGATE.  Indicate that we
         * need a signal, but don't park yet.  Caller will need to
         * retry to make sure it cannot acquire before parking.
         */
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}

private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);
    return Thread.interrupted();
}
```

`shouldParkAfterFailedAcquire `方法主要逻辑是使用`compareAndSetWaitStatus(pred, ws, Node.SIGNAL)`使用CAS将节点状态由 INITIAL 设置成 SIGNAL，表示当前线程阻塞。当 compareAndSetWaitStatus 设置失败则说明 shouldParkAfterFailedAcquire 方法返回 false，然后会在 acquireQueued 方法中死循环中会继续重试，**直至compareAndSetWaitStatus 设置节点状态位为 SIGNAL 时 shouldParkAfterFailedAcquire 返回 true 时才会执行方法 parkAndCheckInterrupt 方法**。（这块在后面研究 AQS 会细讲）

parkAndCheckInterrupt 该方法的关键是会调用 LookSupport.park 方法（关于LookSupport会在以后的文章进行讨论），该方法是用来阻塞当前线程。

所以 acquireQueued 主要做了两件事情：如果当前节点的前驱节点是头节点，并且能够获取独占锁，那么当前线程能够获得锁该方法执行结束退出

如果获取锁失败的话，先将节点状态设置成 SIGNAL，然后调用 LookSupport.park 方法使得当前线程阻塞。

如果 !tryAcquire 和 acquireQueued 都为 true 的话，则打断当前线程。

那么它们的主要流程如下（注：只是加锁流程，并不是 lock 所有流程）

![img](https:////note.youdao.com/src/a88ab09baf32417d9dcf8778f9c8f96c)

**非公平锁的加锁（lock）流程详解**

非公平锁的加锁步骤和公平锁的步骤只有两处不同，一处是非公平锁在加锁前会直接使用 CAS 操作设置同步状态，如果设置成功，就会把当前线程设置为偏向锁的线程；一处是 CAS 操作失败执行 tryAcquire 方法，读取线程同步状态，如果未加锁会使用 CAS 再次进行加锁，不会等待 hasQueuedPredecessors 方法的执行，达到只要线程释放锁就会加锁的目的。下面通过源码和流程图来详细理解

![img](https:////note.youdao.com/src/1143e9cc11c114f47c0f6a53b5e352f3)

这是非公平锁和公平锁不同的两处地方，下面是非公平锁的加锁流程图

![img](https:////note.youdao.com/src/9a1c5b3bfbee730bc5b0af88176e4b82)

**lockInterruptibly 以可中断的方式获取锁**

以下是 JavaDoc 官方解释：

lockInterruptibly 的中文意思为如果没有被打断，则获取锁。如果没有其他线程持有该锁，则获取该锁并立即返回，将锁保持计数设置为1。如果当前线程已经持有锁，那么此方法会立刻返回并且持有锁的数量会 + 1。如果锁是由另一个线程持有的，则出于线程调度目的，当前线程将被禁用，并处于休眠状态，直到发生以下两种情况之一

- 锁被当前线程持有
- 一些其他线程打断了当前线程

如果当前线程获取了锁，则锁保持计数将设置为1。

如果当前线程发生了如下情况：

- 在进入此方法时设置了其中断状态
- 当获取锁的时候发生了中断（Thread.interrupt）

那么当前线程就会抛出InterruptedException 并且当前线程的中断状态会清除。

**下面看一下它的源码是怎么写的**

![img](https:////note.youdao.com/src/e32974bdfeee6dc0b7c4385df22f2ff3)

首先会调用 acquireInterruptibly 这个方法，判断当前线程是否被中断，如果中断抛出异常，没有中断则判断公平锁/非公平锁 是否已经获取锁，如果没有获取锁（tryAcquire 返回 false）则调用 doAcquireInterruptibly 方法，这个方法和 acquireQueued 方法没什么区别，就是线程在等待状态的过程中，如果线程被中断，线程会抛出异常。

下面是它的流程图

![img](https:////note.youdao.com/src/39f2abae6db5916e3d25c4583f2ddb64)

**tryLock 尝试加锁**

仅仅当其他线程没有获取这把锁的时候获取这把锁，tryLock 的源代码和非公平锁的加锁流程基本一致，它的源代码如下

![img](https:////note.youdao.com/src/c488f3c05e4ede7268c8767614562ac7)

tryLock 超时获取锁

ReentrantLock除了能以中断的方式去获取锁，还可以以超时等待的方式去获取锁，所谓超时等待就是线程如果在超时时间内没有获取到锁，那么就会返回false，而不是一直死循环获取。可以使用 tryLock 和 tryLock(timeout, unit)) 结合起来实现公平锁，像这样

if (lock.tryLock() || lock.tryLock(timeout, unit)) {...}

如果超过了指定时间，则返回值为 false。如果时间小于或者等于零，则该方法根本不会等待。

它的源码如下

![img](https:////note.youdao.com/src/a6cfb2111a4c49042ef5c53a65a36b52)

首先需要了解一下 TimeUnit 工具类，TimeUnit 表示给定粒度单位的持续时间，并且提供了一些用于时分秒跨单位转换的方法，通过使用这些方法进行定时和延迟操作。

toNanos 用于把 long 型表示的时间转换成为纳秒，然后判断线程是否被打断，如果没有打断，则以公平锁/非公平锁 的方式获取锁，如果能够获取返回true，获取失败则调用doAcquireNanos方法使用超时等待的方式获取锁。在超时等待获取锁的过程中，如果等待时间大于应等待时间，或者应等待时间设置不合理的话，返回 false。

![img](https:////note.youdao.com/src/d329273b9f93a567f23abcef8f81135b)

这里面以超时的方式获取锁也可以画一张流程图如下

![img](https:////note.youdao.com/src/052098e0f665400a225b1fb3a819a47f)

**unlock 解锁流程**

unlock 和 lock 是一对情侣，它们分不开彼此，在调用 lock 后必须通过 unlock 进行解锁。如果当前线程持有锁，在调用 unlock 后，count 计数将减少。如果保持计数为0就会进行解锁。如果当前线程没有持有锁，在调用 unlock 会抛出 IllegalMonitorStateException 异常。下面是它的源码

![img](https:////note.youdao.com/src/d14fd9f0c17260347b0c0fe6ac17858a)

在有了上面阅读源码的经历后，相信你会很快明白这段代码的意思，锁的释放不会区分公平锁还是非公平锁，主要的判断逻辑就是 tryRelease 方法，getState 方法会取得同步锁的重入次数，如果是获取了偏向锁，那么可能会多次获取，state 的值会大于 1，这时候 c 的值 > 0 ，返回 false，解锁失败。如果 state = 1，那么 c = 0，再判断当前线程是否是独占锁的线程，释放独占锁，返回 true，当 head 指向的头结点不为 null，并且该节点的状态值不为0的话才会执行 unparkSuccessor 方法，再进行锁的获取。

![img](https:////note.youdao.com/src/0cf99b12dbc1076a8397355c286e9730)

**ReentrantLock 其他方法**

**isHeldByCurrentThread & getHoldCount**

在多线程同时访问时，ReentrantLock 由最后一次成功锁定的线程拥有，当这把锁没有被其他线程拥有时，线程调用 lock() 方法会立刻返回并成功获取锁。如果当前线程已经拥有锁，这个方法会立刻返回。可以通过 isHeldByCurrentThread 和 getHoldCount 来进行检查。

首先来看 isHeldByCurrentThread 方法

public boolean isHeldByCurrentThread() {

  return sync.isHeldExclusively();

}

根据方法名可以略知一二，是否被当前线程持有，它用来询问锁是否被其他线程拥有，这个方法和 Thread.holdsLock(Object) 方法内置的监视器锁相同，而 Thread.holdsLock(Object) 是 Thread 类的静态方法，是一个 native 类，它表示的意思是如果当前线程在某个对象上持有 monitor lock(监视器锁) 就会返回 true。这个类没有实际作用，仅仅用来测试和调试所用。例如

private ReentrantLock lock = new ReentrantLock();

public void lock(){

  assert lock.isHeldByCurrentThread();

}

这个方法也可以确保重入锁能够表现出不可重入的行为

private ReentrantLock lock = new ReentrantLock();

public void lock(){

  assert !lock.isHeldByCurrentThread();

  lock.lock();

  try {

​    // 执行业务代码

  }finally {

​    lock.unlock();

  }

}

如果当前线程持有锁则 lock.isHeldByCurrentThread() 返回 true，否则返回 false。

我们在了解它的用法后，看一下它内部是怎样实现的，它内部只是调用了一下  sync.isHeldExclusively()，sync 是 ReentrantLock 的一个静态内部类，基于 AQS 实现，而 AQS 它是一种抽象队列同步器，是许多并发实现类的基础，例如 **ReentrantLock/Semaphore/CountDownLatch**。sync.isHeldExclusively() 方法如下

protected final boolean isHeldExclusively() {

  return getExclusiveOwnerThread() == Thread.currentThread();

}

此方法会在拥有锁之前先去读一下状态，如果当前线程是锁的拥有者，则不需要检查。

getHoldCount()方法和isHeldByCurrentThread 都是用来检查线程是否持有锁的方法，不同之处在于 getHoldCount() 用来查询当前线程持有锁的数量，对于每个未通过解锁操作匹配的锁定操作，线程都会保持锁定状态，这个方法也通常用于调试和测试，例如

private ReentrantLock lock = new ReentrantLock();

public void lock(){

  assert lock.getHoldCount() == 0;

  lock.lock();

  try {

​    // 执行业务代码

  }finally {

​    lock.unlock();

  }

}

这个方法会返回当前线程持有锁的次数，如果当前线程没有持有锁，则返回0。

**newCondition 创建 ConditionObject 对象**

ReentrantLock 可以通过 newCondition 方法创建 ConditionObject 对象，而 ConditionObject 实现了 Condition  接口，关于 Condition 的用法我们后面再讲。

**isLocked 判断是否锁定**

查询是否有任意线程已经获取锁，这个方法用来监视系统状态，而不是用来同步控制，很简单，直接判断 state 是否等于0。

**isFair 判断是否是公平锁的实例**

这个方法也比较简单，直接使用 instanceof 判断是不是 FairSync 内部类的实例

public final boolean isFair() {

  return sync instanceof FairSync;

}

**getOwner 判断锁拥有者**

判断同步状态是否为0，如果是0，则没有线程拥有锁，如果不是0，直接返回获取锁的线程。

final Thread getOwner() {

  return getState() == 0 ? null : getExclusiveOwnerThread();

}

**hasQueuedThreads 是否有等待线程**

判断是否有线程正在等待获取锁，如果头节点与尾节点不相等，说明有等待获取锁的线程。

public final boolean hasQueuedThreads() {

  return head != tail;

}

**isQueued 判断线程是否排队**

判断给定的线程是否正在排队，如果正在排队，返回 true。这个方法会遍历队列，如果找到匹配的线程，返回true

public final boolean isQueued(Thread thread) {

  if (thread == null)

​    throw new NullPointerException();

  for (Node p = tail; p != null; p = p.prev)

​    if (p.thread == thread)

​      return true;

  return false;

}

**getQueueLength 获取队列长度**

此方法会返回一个队列长度的估计值，该值只是一个估计值，因为在此方法遍历内部数据结构时，线程数可能会动态变化。 此方法设计用于监视系统状态，而不用于同步控制。

public final int getQueueLength() {

  int n = 0;

  for (Node p = tail; p != null; p = p.prev) {

​    if (p.thread != null)

​      ++n;

  }

  return n;

}

**getQueuedThreads 获取排队线程**

返回一个包含可能正在等待获取此锁的线程的集合。 因为实际的线程集在构造此结果时可能会动态更改，所以返回的集合只是一个大概的列表集合。 返回的集合的元素没有特定的顺序。

public final Collection<Thread> getQueuedThreads() {

  ArrayList<Thread> list = new ArrayList<Thread>();

  for (Node p = tail; p != null; p = p.prev) {

​    Thread t = p.thread;

​    if (t != null)

​      list.add(t);

  }

  return list;

}

**回答上面那个问题**

那么你看完源码分析后，你能总结出 synchronized 和 lock 锁的实现 ReentrantLock 有什么异同吗？

Synchronzied 和 Lock 的主要区别如下：

- **存在层面**：Syncronized 是Java 中的一个关键字，存在于 JVM 层面，Lock 是 Java 中的一个接口
- **锁的释放条件**：1. 获取锁的线程执行完同步代码后，自动释放；2. 线程发生异常时，JVM会让线程释放锁；Lock 必须在 finally 关键字中释放锁，不然容易造成线程死锁
- **锁的获取**: 在 Syncronized 中，假设线程 A 获得锁，B 线程等待。如果 A 发生阻塞，那么 B 会一直等待。在 Lock 中，会分情况而定，Lock 中有尝试获取锁的方法，如果尝试获取到锁，则不用一直等待
- **锁的状态**：Synchronized 无法判断锁的状态，Lock 则可以判断
- **锁的类型**：Synchronized 是可重入，不可中断，非公平锁；Lock 锁则是 可重入，可判断，可公平锁
- **锁的性能**：Synchronized 适用于少量同步的情况下，性能开销比较大。Lock 锁适用于大量同步阶段：

Lock 锁可以提高多个线程进行读的效率(使用 readWriteLock)

- 在竞争不是很激烈的情况下，Synchronized的性能要优于ReetrantLock，但是在资源竞争很激烈的情况下，Synchronized的性能会下降几十倍，但是ReetrantLock的性能能维持常态；
- ReetrantLock 提供了多样化的同步，比如有时间限制的同步，可以被Interrupt的同步（synchronized的同步是不能Interrupt的）等

**还有什么要说的吗**

面试官可能还会问你 ReentrantLock 的加锁流程是怎样的，其实如果你能把源码给他讲出来的话，一定是高分。如果你记不住源码流程的话可以记住下面这个**简化版的加锁流程**

- 如果 lock 加锁设置成功，设置当前线程为独占锁的线程；
- 如果 lock 加锁设置失败，还会再尝试获取一次锁数量，

如果锁数量为0，再基于 CAS 尝试将 state（锁数量）从0设置为1一次，如果设置成功，设置当前线程为独占锁的线程；

如果锁数量不为0或者上边的尝试又失败了，查看当前线程是不是已经是独占锁的线程了，如果是，则将当前的锁数量+1；如果不是，则将该线程封装在一个Node内，并加入到等待队列中去。等待被其前一个线程节点唤醒。