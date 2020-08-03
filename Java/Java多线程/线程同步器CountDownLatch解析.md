## CountDownLatch原理探究

### 使用方法

平时开发过程中会遇到需要在主线程中开启多个线程去并行执行任务，且主线程需要等待所有子线程执行完毕后再进行汇总的场景。CountDownLatch出现前都是使用线程的`join()`方法，但不够灵活，且平时大多使用线程池来执行我们的Runnable，根本不能调用线程的`join()`方法，而CountDownLatch则满足了我们的需求。

例如我们想让两个线程执行完毕后，返回到主线程再执行其他任务，实例代码如下：

```java
public class JoinCountDownLatch {

    private static CountDownLatch countDownLatch = new CountDownLatch(2);

    public static void main(String[] args) throws InterruptedException {
        ExecutorService executorService = Executors.newFixedThreadPool(2);
        executorService.submit(new Runnable() {
            @Override
            public void run() {
                try {
                    Thread.sleep(1000);
                    System.out.println("Child ThreadOne over!");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    countDownLatch.countDown();   //线程A执行完后计数减1
                }
            }
        });

        executorService.submit(new Runnable() {
            @Override
            public void run() {
                try {
                    Thread.sleep(1000);
                    System.out.println("Child ThreadTwo over!");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    countDownLatch.countDown();   //线程B执行完后计数减1
                }
            }
        });
        System.out.println("wait all child thread over!");
        countDownLatch.await();      // 一直阻塞，直到子线程执行完(计数器为0)
        System.out.println("all child thread over!");
        executorService.shutdown();  // 关闭线程池
    }
}
```

说一下CountDownLatch与`join()`方法区别，

- 调用一个子线程的`join()`方法后，该线程会一直被阻塞直到子线程运行完毕；而CountDownLatch则使用计数器来允许子线程执行完毕或在运行中递减计数，**即CountDownLatch可以在子线程运行的任何时候让`await()`方法返回而不一定必须要等到线程结束**。
- 使用线程池管理线程时一般是直接添加Runnable到线程池，这时无法使用线程的join方法；而CountDownLatch则可以使用，相比于join方法更加灵活。

### 实现原理探究

![img](https://note.youdao.com/yws/public/resource/56347de0c9d0b624bcbb02366ca7a2b8/xmlnote/BFC6389A3F714436BF57D7849C871CE3/D0D70C3128694F9398F1109478632C1D/11179)

CountDownLatch是使用AQS实现的，它实际上是把计数器的值赋给了AQS的状态变量`state`。然后我们看一下其中的几个方法实现原理

```java
public CountDownLatch(int count) {
    if (count < 0) throw new IllegalArgumentException("count < 0");
    this.sync = new Sync(count);
}

Sync(int count) {
    setState(count);
}
```

#### void await()方法

当线程调用此方法后会被阻塞，直到下面的情况之一发生才会返回：

- 所有线程都调用了CountDownLatch对象的`countDown()`方法后，**也就是计数器值为0时**
- **其他线程调用当前线程的interrupt()方法中断当前线程**，当前线程抛出InterruptedException异常

```java
public void await() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}

public final void acquireSharedInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted())  //先看线程是否被中断，是则抛出异常
        throw new InterruptedException();
    //查看当前计数器值是否为0，为0直接返回，否则进入AQS队列等待
    if (tryAcquireShared(arg) < 0)  
        doAcquireSharedInterruptibly(arg); //这个函数要了解
}

// Sync类实现的AQS接口
protected int tryAcquireShared(int acquires) {
    return (getState() == 0) ? 1 : -1;  //直接判断state值
}
```

通过上面代码可知，该方法特点是线程获取资源时可被中断，且获取的资源是**共享资源**。能看到tryAcquiredShared传递的arg参数没有被用到，此方法只是为了检查当前状态值是不是为0。

#### boolean await(long timeout, TimeUnit unit)方法

此方法就是相对于上一个方法，多加了一个超时设置。**当超过一定时间后会自动返回**。

```
public boolean await(long timeout, TimeUnit unit)
    throws InterruptedException {
    return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
}
```

#### void countDown()方法

线程调用该方法后计数器的值递减，**递减后如果计数器值为0则唤醒所有因调用await方法而被阻塞的线程**，否则什么都不做。

```java
public void countDown() {
    sync.releaseShared(1);
}
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {  //计数器为0
        doReleaseShared(); 				//若状态量为0，则唤醒所有阻塞线程
        return true;
    }
    return false;
}

protected boolean tryReleaseShared(int releases) {
    // 不停CAS将计数器减1，当为0时唤醒所有阻塞线程
    for (;;) {
        int c = getState();
        // 如果计数器本身为0，则不能release，返回false
        // 是为了防止计数器为0后，还有线程调用此方法导致state为负数
        if (c == 0)  
            return false;
        int nextc = c-1;  //计数器减1，并尝试CAS
        if (compareAndSetState(c, nextc))
            return nextc == 0;  //查看计数器是否为0
    }
}

private void doReleaseShared() {
    
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) { //此线程阻塞，在等待通知
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                unparkSuccessor(h); //释放线程
            }
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        if (h == head)                   // loop if head changed
            break;
    }
}
```

上面看出是调用AQS的doReleaseShared方法来激活阻塞的线程。

## 回环屏障CyclicBarrier原理探究

上面说到CountDownLatch，但此计数器是个一次性的，也就是等到计数器值变为0后再调用CountDownLatch的await()和countDown()方法都会立刻返回。所以为此CyclicBarrier诞生了，它可以让一组线程全部到达一个状态后再全部同时执行。

- **回环：**之所以叫回环是因为当所有等待线程执行完毕后，并重置CyclicBarrier的状态后它可以被重用。
- **屏障：**之所有叫屏障是因为线程调用`await()`方法后就会被阻塞，这个阻塞点就被称为屏障点，等所有线程都调用了await()方法后，线程们就会冲破屏障继续向下运行。

### 使用方法

假设我们要使用两个线程去执行一个被分解的任务A，当两个线程把自己的任务都执行完毕后再对它们的结果进行汇总处理。

```java
public class CycleBarrierTest1 {
    private static CyclicBarrier cyclicBarrier 
        = new CyclicBarrier(2, new Runnable() {
        @Override
        public void run() { //到达屏障点后执行的方法
            System.out.println(Thread.currentThread() 
                + " task1 merge result");
        }
    });

    public static void main(String[] args) {
        ExecutorService executorService = 
            Executors.newFixedThreadPool(2);
        executorService.submit(new Runnable() {
            @Override
            public void run() {
                try {
                    System.out.println(Thread.currentThread() 
                            + " task-1");
                    System.out.println(Thread.currentThread() 
                            + " enter in barrier");
                    cyclicBarrier.await(); //计数器不为0，阻塞
                    System.out.println(Thread.currentThread() 
                            + " enter out barrier");
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        });

        executorService.submit(new Runnable() {
            @Override
            public void run() {
                try {
                    System.out.println(Thread.currentThread() 
                            + " task1-2");
                    System.out.println(Thread.currentThread() 
                            + " enter in barrier");
                    cyclicBarrier.await(); //计数器不为0，阻塞
                    System.out.println(Thread.currentThread() 
                            + " enter out barrier");
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        });
        executorService.shutdown();
    }
}
```

上面代码中，线程池的两个线程执行过程中，若调用`cyclicBarrier.await()`方法后，计数器值会减1。当两个线程都调用了此方法后，计数器会变为0，此时就会去执行CyclicBarrier构造函数中的任务，任务完毕后退出屏障点并且唤醒被阻塞的线程。

上面例子说明了多个线程之间是相互等待的，假设计数器值为N，则随后调用的`await()`方法的N-1个线程都会等待第N个线程执行完await后，计数器变为0，他才会发出通知唤醒前面的N-1个线程。也就是当全部线程到达屏障点后才能一块继续向下执行。

### 实现原理探究

![img](https://note.youdao.com/yws/public/resource/56347de0c9d0b624bcbb02366ca7a2b8/xmlnote/BFC6389A3F714436BF57D7849C871CE3/AE91CF4FD8E14CEEBD1C79B23F188C51/11343)

能看到**CyclicBarrier基于独占锁实现**，本质底层还是基于AQS。**parties**用来记录线程个数，这里表示多少线程调用`await()`后，所有线程才会冲破屏障继续运行。而count一开始等于parties，每当有线程调用await方法就递减1，当count为0时就表示所有线程都到了屏障点。**由于CyclicBarrier是可以被复用的，所以使用了parties和count两个变量，当count为0后会将parties的值赋给count从而进行复用**。

```java
public CyclicBarrier(int parties, Runnable barrierAction) {
    if (parties <= 0) 
      throw new IllegalArgumentException();
    this.parties = parties;   // 重点是多传了一个parties变量
    this.count = parties;
    this.barrierCommand = barrierAction;
}
```

还有一个变量barrierCommand也通过构造函数传递，它是一个任务，**它的执行时机是当所有线程都到达屏障点后**。使用lock首先保证了更新计数器count的原子性，另外使用lock的**条件变量trip**支持线程间使用await和signal操作进行同步。

最后在变量generation内部有一个变量broken，它是个boolean类型的值，**用来记录当前屏障是否被打破。注意此处的broken没有声明为volatile，因为是在锁内使用变量，所以不需要声明**。

```java
private static class Generation {
    boolean broken = false;
}
```

#### int await()方法

当线程调用CyclicBarrier的该方法时会被阻塞，直到下面条件之一满足才会返回：

- parties个线程都调用了`await()`方法，也就是线程都到了屏障点
- 其他线程调用当前线程的`interrupt()`方法中断当前线程，当前线程抛出InterruptedException异常
- 与当前屏障点关联的Generation对象的broken标志被设置为**true**时，会抛出BrokenBarrierException异常，然后返回。
- 类似的有个**await(long timeout, TimeUnit unit)**方法，只是在上面基础上加了个超时自动返回

此方法在内部调用了dowait方法，第一个参数为false则说明不设置超时时间，此时第二个参数没用；第一个参数为true则设置超时时间，第二个参数为超时时间。

```java
public int await() throws InterruptedException, BrokenBarrierException {
    try {
        return dowait(false, 0L);
    } catch (TimeoutException toe) {
        throw new Error(toe); // cannot happen
    }
}
```

#### int dowait(boolean timed, long nanos)方法

该方法实现了CyclicBarrier的核心功能，其代码如下：

```java
private int dowait(boolean timed, long nanos) throws Exception  {
    final ReentrantLock lock = this.lock; 
    lock.lock();    // 先获得全局锁，然后上锁
    try {
        final Generation g = generation;
        if (g.broken)  // 若当前屏障已经被打破则返回异常
            throw new BrokenBarrierException();
        if (Thread.interrupted()) { // 若线程被中断直接返回异常
            breakBarrier();					//打破屏障
            throw new InterruptedException();
        }
        int index = --count;
        // 若为0说明所有线程都到了屏障点，执行初始化时传递的任务
        if (index == 0) {  
            boolean ranAction = false;
            try {
                final Runnable command = barrierCommand;
                if (command != null)
                    command.run(); //执行任务
                ranAction = true;
                //激活其他因调用await()而阻塞的线程，重置CyclicBarrier
                nextGeneration(); 
                return 0;
            } finally {
                if (!ranAction) //任务没有执行成功，则直接打破屏障
                    breakBarrier();
            }
        }

        // 若index!=0说明，则会执行await阻塞自己
        for (;;) {
            try {
                if (!timed) //没设置过期时间，调用await阻塞自己
                    trip.await();
                else if (nanos > 0L)
                    nanos = trip.awaitNanos(nanos);
            } catch (InterruptedException ie) {
                //...
            }

            if (g.broken)
                throw new BrokenBarrierException();

            if (g != generation)
                return index;

            if (timed && nanos <= 0L) {
                breakBarrier();
                throw new TimeoutException();
            }
        }
    } finally {
        lock.unlock();
    }
}

private void breakBarrier() {
    generation.broken = true;
    count = parties;
    trip.signalAll();
}

private void nextGeneration() {
    // 唤醒所有的阻塞线程
    trip.signalAll();
    // 恢复count计数器，重置屏障打破标记
    count = parties;
    generation = new Generation();
}
```

## 信号量Semaphore原理探究

与CountDownLatch和CyclicBarrier不同的是，它内部的计数器是**递增**的，并且一开始初始化Semaphore时可执行一个初始值，但并不需要知道需要同步的线程个数，而是在需要同步的地方调用`acquire()`方法时指定需要同步的线程个数。

### 使用方法

```java
public class SemaphoreTest {
    // 创建一个信号量，构造参数为0说明当前信号量计数器的值为0
    private static Semaphore semaphore = new Semaphore(0);

    public static void main(String[] args) throws InterruptedException {
        ExecutorService executorService = 
            Executors.newFixedThreadPool(2);

        executorService.submit(new Runnable() {
            @Override
            public void run() {
                try {
                    System.out.println(Thread.currentThread() + " over");
                    semaphore.release();
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        });

        executorService.submit(new Runnable() {
            @Override
            public void run() {
                try {
                    System.out.println(Thread.currentThread() + " over");
                    semaphore.release();
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        });
        //传的参数为2，说明信号量变为2时才返回
        semaphore.acquire(2);
        System.out.println("all child thread over!");
        executorService.shutdown();
    }
}
```

所以到这里就明白了，若构造Semaphore时传递的参数为N，并在M个线程中调用了该信号量的`release()`方法，那么在调用`acquire()`使M个线程同步时传递的参数应该是M+N。也就是说**调用acquire方法的线程一直阻塞，直到信号量数目变为指定传入的数字为止**。

### 实现原理探究

![img](https://note.youdao.com/yws/public/resource/56347de0c9d0b624bcbb02366ca7a2b8/xmlnote/BFC6389A3F714436BF57D7849C871CE3/ADA8BFBD4C1D4CA59EB9DB3D5E9D08D7/11482)

Semaphore还是使用AQS实现，Sync只是对AQS的一个修饰且Sync有两个实现类：用来指定获取信号量时是否采用公平策略。Semaphore默认采用**非公平策略**，同样此处的permits也表示信号量个数，赋给AQS的state变量。

```java
public Semaphore(int permits) {
    sync = new NonfairSync(permits);
}
// 可设置为公平获取信号量
public Semaphore(int permits, boolean fair) {
    sync = fair ? new FairSync(permits) : new NonfairSync(permits);
}

Sync(int permits) {
    setState(permits);
}
```

#### void acquire()方法

线程调用该方法目的是希望获取一个信号量资源。

- 若当前信号量个数大于0，则当前信号量计数会减1然后该方法直接返回。
- 若当前信号量个数等于0，则当前线程会被放入AQS的阻塞队列。其它线程调用当前线程的interrupr()方法中断本线程时，本线程会抛出InterruptedException异常返回。

```java
public void acquire() throws InterruptedException {
    sync.acquireSharedInterruptibly(1); //尝试获取1个信号量
}
public final void acquireSharedInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted()) //若线程中断，则返回异常
        throw new InterruptedException();
    if (tryAcquireShared(arg) < 0) //尝试获取信号量，由子类实现
        doAcquireSharedInterruptibly(arg);
}
```

其中的`tryAcquireShared()`是由Sync的子类实现的，所以从两方面来讨论。先说一下非公平策略：

```java
// 非公平策略
protected int tryAcquireShared(int acquires) {
    return nonfairTryAcquireShared(acquires);
}
final int nonfairTryAcquireShared(int acquires) {
    for (;;) {
        int available = getState(); //得到信号量，且减去acquires
        int remaining = available - acquires;
        //若剩余值小于0则直接返回负数，否则使用CAS设置新的剩余量并返回
        if (remaining < 0 ||  
            compareAndSetState(available, remaining))
            return remaining;
    }
}
```

此策略是先获得信号量，并减去需要获取的值得到剩余信号量个数。

- 若剩余值小于0，说明当前信号量个数满足不了需求，返回负数。且当前线程会被会执行后续的doAcquireSharedInterruptibly而放到AQS的阻塞队列中被挂起
- 若剩余值大于0，则使用CAS设置当前信号量为剩余值，然后返回剩余值。

```java
// 公平策略
protected int tryAcquireShared(int acquires) {
    for (;;) {
        //先判断前面有没有其他线程，有则直接返回-1，从而被挂起
        if (hasQueuedPredecessors()) 
            return -1;
        int available = getState();
        int remaining = available - acquires;
        if (remaining < 0 ||
            compareAndSetState(available, remaining))
            return remaining;
    }
}
```

公平策略只是先判断它的前面（阻塞队列中）有没有其它线程，若有则返回负数，从而执行后续的doAcquireSharedInterruptibly而放到AQS的阻塞队列中被挂起。

#### void acquire(int permits)方法

它与`acquire()`方法不同，后者只需要获取1个信号量值，而前者则获取permits个。

```java
public void acquire(int permits) throws InterruptedException {
    if (permits < 0) throw new IllegalArgumentException();
    sync.acquireSharedInterruptibly(permits);
}
```

#### void acquireUninterruptibly(int permits)方法、不带permits参数的此方法

该方法与acquire()类似，不同之处在于该方法对中断不响应。

```java
public void acquireUninterruptibly(int permits) {
    if (permits < 0) throw new IllegalArgumentException();
    sync.acquireShared(permits); //根据具体实现，如是否公平来做决定
}
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
}
```

#### void release()方法、带permits参数的此方法

该方法作用是把当前Semaphore对象的信号量值加1，若当前有线程因为调用acquire方法而被放入了AQS的阻塞队列，则会根据**公平策略**选择一个信号量个数能被满足线程进行激活，激活的线程会尝试获取刚增加的信号量。

```java
public void release() {
    sync.releaseShared(1);
}

public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) { //使用CAS尝试更改信号量
        doReleaseShared();//唤醒AQS里一个因调用park而被挂起的线程
        return true;
    }
    return false;
}

protected final boolean tryReleaseShared(int releases) {
    for (;;) {
        int current = getState();
        int next = current + releases;
        if (next < current) // 数字溢出了
            throw new Error("Maximum permit count exceeded");
        if (compareAndSetState(current, next))
            return true;
    }
}
```

能看到该信号量是线程共享的，信号量没有和固定线程绑定，多个线程可以同时使用CAS去更新信号量的值而不会被阻塞。