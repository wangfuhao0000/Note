线程池的出现为了解决两个问题：一是**性能问题**。因为线程的创建和销毁都是需要开销的，所以可以复用线程池里的线程，从而不需要每次执行异步任务都重新创建和销毁线程。二是线程池提供了一种**资源限制和管理**的手段，比如可限制线程个数或动态增加线程等。

`Executors`的工厂方法提供了许多种线程池（静态方法），从而满足不同需求，比如`newCachedThreadPool`（线程池线程个数最多可达Integer.MAX_VALUE，线程自动回收）、`newFixedThreadPool`（固定大小的线程池）和`newSingleThreadExecutor`（单个线程）等来创建线程池。

### 1.ThreadPoolExecutor类解析

首先说一下里面的一个**AtomicInteger的原子变量ctl**，它用来同时记录**线程池状态**和线程池中**线程个数**：其中高3位用来表示线程池状态，后面29位用来记录线程池线程个数（假设int为32位）。线程池状态有多个，当然它们对应的值可以去源码里寻找，无非就是针对高三位有不同设置，主要说一下各状态的含义（数值是从小到大排列的）：

- RUNNING：接受新任务并且处理阻塞队列里的任务。
- SHUTDOWN：拒绝新任务但是处理阻塞队列里的任务。
- STOP：拒绝新任务并且抛弃阻塞队列里的任务，同时中断正在处理的任务。
- TIDYING：所有任务都执行完（包括阻塞队列里的任务）后**当前线程池活动线程数为0**，将要调用terminated方法。
- TERMINATED：终止状态。terminated方法调用完以后的状态。

线程池就是在这几种状态之间不停的转换，其转换图如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190915101343615.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3JlYWNod2FuZw==,size_16,color_FFFFFF,t_70)

#### 线程参数

线程池有各种各样的参数，如下：

- corePoolSize：线程池核心线程个数。
- workQueue：用于保存等待执行的任务的**阻塞队列**，比如基于数组的有界ArrayBlockingQueue、基于链表的无界LinkedBlockingQueue、最多只有一个元素的同步队列SynchronousQueue和优先级队列PriorityBlockingQueue等。
- maximunPoolSize:线程池最大线程数量。
- ThreadFactory：创建线程的工厂。
- RejectedExecutionHandler：**饱和策略**，队列满且线程个数达到maximunPoolSize后采取的策略，如AbortPolicy（抛出异常）、CallerRunsPolicy（使用调用者所在线程运行任务）、DiscardOldestPolicy（调用poll丢弃一个任务，执行当前任务）及DiscardPolicy（默认丢弃，不抛出异常）
- keepAliveTime：存活时间。若当前线程池中的线程数量比核心线程数量多，并且是闲置状态，则这些闲置的线程能存活的最大时间。也就是把超过核心线程数量的线程看成是“多余”的，要设一个过期日期。
- TimeUnit：存活时间的时间单位。

#### 线程池类型

- newFIxedThreadPool：创建一个核心线程个数和最大线程个数都为`nThreads`的线程池，并且阻塞队列长度为**Integer.MAX_VALUE**。keepAliveTime=0说明只要线程个数比核心线程个数多并且当前空闲则回收。

```java
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());
}
//使用自定义线程创建工厂
public static ExecutorService newFixedThreadPool(int nThreads, ThreadFactory threadFactory) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>(),
                                  threadFactory);
```

- newSingleThreadExecutor：创建一个核心线程个数和最大线程个数都为1的线程池，并且阻塞队列长度为**Integer.MAX_VALUE**。keepAliveTime=0说明只要线程个数比核心线程个数多并且当前空闲则回收。

```java
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}

public static ExecutorService newSingleThreadExecutor(ThreadFactory threadFactory) {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>(),
                                threadFactory));
}
```

- newCachedThreadPool：创建一个按需创建线程的线程池，初始线程个数为0，最多线程个数为Integer.MAX_VALUE。并且阻塞队列为同步队列。keepAliveTime=60说明只要当前线程在60s内空闲则回收。这个类型特殊之处在于：**加入同步队列的任务会被马上执行，同步队列里面最多只有一个任务**。

```java
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
}

 public static ExecutorService newCachedThreadPool(ThreadFactory threadFactory) {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>(),
                                  threadFactory);
}
```

同时有一个变量`mainLock`是独占锁，用来控制新增Worker线程操作的原子性。termination是该锁对应的条件队列，在线程调用awaitTermination时，用来存放阻塞的线程。

Worker集成AQS和Runnable接口，是具体承载任务的对象。Worker继承了AQS，自己实现了简单不可重入独占锁，其中state=0表示锁未被获取状态，state=1表示锁已经被获取的状态，state=-1是创建Worker时默认的状态，创建时设置为-1是为了避免该线程在运行runWorker()方法前被中断。后面会具体讲到Worker类

### 2.ThreadPoolExecutor代码解析

#### 2.1 execute(Runnable command)方法

这个方法就是将一个任务放入到线程池中，它主要包括4个步骤：

1. **如果当前运行线程少于corePoolSize，则创建新线程执行任务（这一步需要获取全局锁）。**
2. **如果运行的线程等于或多于corePoolSize，则将任务加入BlockingQueue。**
3. **如果无法将任务加入BlockingQueue（队列已满），则创建新的线程来处理任务（也需要获取全局锁）**
4. **若创建新线程将使得当前线程数超过maximunPoolSIze，任务被拒绝并调用RejectedExecutionHandler.rejectedExecution()方法**

代码如下：

```java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    int c = ctl.get();
    //1.如果当前运行的线程少于corePoolSize，则开启新线程运行
    if (workerCountOf(c) < corePoolSize) {	
        if (addWorker(command, true))
            return;
        c = ctl.get();		//添加失败重新获取一次线程池状态
    }
  
    //2.否则如果线程池处于RUNNING状态，则添加到任务阻塞队列中
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        //3.若添加后线程池状态不是RUNNING，则是要抛弃新任务的，所以删除此任务并执行拒绝策略
        if (! isRunning(recheck) && remove(command))
            reject(command);
         //4.如果当前线程池为空，则添加一个线程来处理阻塞队列中的任务
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    //5.如果队列也满了，则新增线程，新增失败则执行拒绝策略
    else if (!addWorker(command, false))
        reject(command);
}
```

#### 2.2 addWorker(Runnable firstTask, boolean core)方法

接下来再看一下新增线程的`addWorker()`方法，代码如下：

```java
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // 1.检查队列为空是不是只是因为没有任务了
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
               firstTask == null &&
               ! workQueue.isEmpty()))
            return false;
  			//2.循环CAS增加线程个数
        for (;;) {
            int wc = workerCountOf(c);
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize))
                return false;
             // 2.1 CAS增加线程个数，同时只有一个线程成功，成功直接退出循环
            if (compareAndIncrementWorkerCount(c))
                break retry;
             // 2.2 CAS失败则重新获取状态，若状态变了则从外部循环重新开始，否则直接在内层循环中进行CAS
            c = ctl.get();  // Re-read ctl
            if (runStateOf(c) != rs)
                continue retry;
            // else CAS failed due to workerCount change; retry inner loop
        }
    }
		// 3. CAS 成功了，即数量参数更新成功
    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {
      // 3.1 创建一个新的worker（线程池的基本单位）
        w = new Worker(firstTask);
        final Thread t = w.thread;
        if (t != null) {
            final ReentrantLock mainLock = this.mainLock;
            //3.2 加锁，保证添加worker时的原子性，即只有一个线程能将新建的线程加到池中
            mainLock.lock();
            try {
                // Recheck while holding lock.
                // Back out on ThreadFactory failure or if
                // shut down before lock acquired.
                int rs = runStateOf(ctl.get());
      					//3.3 检查队列为空只是因为没有任务了
                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                    if (t.isAlive()) // precheck that t is startable
                        throw new IllegalThreadStateException();
                     // 3.4 添加任务
                    workers.add(w);
                    int s = workers.size();
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    workerAdded = true;
                }
            } finally {
                mainLock.unlock();
            }
            // 3.5 添加成功后则启动任务
            if (workerAdded) {
                t.start();
                workerStarted = true;
            }
        }
    } finally {
        if (! workerStarted)
            addWorkerFailed(w);
    }
    return workerStarted;
}
```

代码比较长。首先我们要知道的就是所谓的线程池，里面包含的线程就是这些Worker。上面的代码主要分为两部分：

- 第一部分的大的for循环就是用来原子地改变`workerCount`的，这是一个**不停进行CAS的过程**
- 第二部分则是新建Worker，并将它加入到池中（workers，它是一个`HashSet`），**这个过程涉及到了向线程池加线程，因此需要加锁**。

添加以后就会执行3.5处的start()，这会启动这个worker，让其不断地获取自己里面的firstTask，或者从阻塞队列中取task，这个在后面讲。

这个地方我们看一下代码1和3.3两处，两者类似。3.3处几乎相当于1处相反的情况，或者说3.3处的更容易理解一些。就是检查线程池是不是正处于运行状态（此时可以添加新线程）或者线程池处于shutdown状态，但是是因为没有能执行的线程了（firstTask == null）并且阻塞队列里还有任务（! worker.isEmpty()），那么此时也可以添加一个新的线程。

当然也可以对代码1进行等价运算，展开后半部分的！则会有三种情况使得添加过程返回`false`，如下：

```java
s >= SHUTDOWN &&
	(rs != SHUTDOWN ||  //(1)当前线程池状态为STOP、TIDYING或TERMINATED
	firstTask != null ||  //(2)当前线程池状态为SHUTDOWN并且已经有了第一个任务（能执行任务的线程）
	workQueue.isEmpty())  //(3)当前线程池状态为SHUTDOWN并且任务队列为空
1234
```

所以总体来说就两部分：**改变workerCount**和**创建新worker并启动**。

#### 2.3 Worker类

首先我们先看一下所谓的线程池中的基本单位，即Worker类，它的定义及构造函数如下：

```java
private final class Worker
    extends AbstractQueuedSynchronizer
    implements Runnable
{
    ...

    final Thread thread;

    Runnable firstTask;

    volatile long completedTasks;   // 当前线程完成的任务个数

    Worker(Runnable firstTask) {
        setState(-1); // inhibit interrupts until runWorker
        this.firstTask = firstTask;
        // 将当前对象作为参数传到它的thread变量中，这样在执行thread的start方法时，便执行worker的run方法。
        this.thread = getThreadFactory().newThread(this);
    }

    //将执行的过程交给外部方法来处理
    public void run() {
        runWorker(this);
    }

    ...
}
```

里面有一个`thread`变量，这个变量便存放了真正去执行任务的线程。还有一个task表示这个worker当前正在执行的任务。特别要注意的是构造函数的第三行，它将自己作为对象传到了thread变量的构造中，**这样在执行worker.thread.start()方法时，其实就是执行了worker.run()方法，进而调用了外部的runWorker方法**。这也就对应了上一小节中addWorker方法里面3.4处调用的是worker.thread.start()方法来启动这个worker。

#### 2.4 runWorker(Worker w)方法

`runWorker(Worker w)`方法通过将当前自己（某个worker）作为参数传进去来启动自己worker的工作流程，让这个worker不停地检查自己的task或者从阻塞队列中取task。runWorker方法如下：

```java
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();		//获取当前线程，方便它中断worker的执行
    Runnable task = w.firstTask;
    w.firstTask = null;		// 设置为空后方便它从阻塞队列中取后续的任务并进行赋值
    w.unlock(); 					// 允许中断
    boolean completedAbruptly = true;
    try {
        while (task != null || (task = getTask()) != null) {
            w.lock();			//1. 每次执行一个任务时要加锁
            ...
            // If pool is stopping, ensure thread is interrupted;
            // if not, ensure thread is not interrupted.  This
            // requires a recheck in second case to deal with
            // shutdownNow race while clearing interrupt
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() &&
                  runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            try {
                beforeExecute(wt, task);			//2. 执行前做的事
                Throwable thrown = null;
                try {
                    task.run();						//3. 真正执行任务
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    afterExecute(task, thrown);			//4. 执行后做的一些事
                }
            } finally {
                task = null;
                w.completedTasks++;	//这个线程所做的工作数量加1
                w.unlock();		//5. 执行完毕后统计当前worker完成任务数释放锁
            }
        }
        completedAbruptly = false;
    } finally {			//6. 循环结束，这个worker取不到task了，执行自身的清理工作
        processWorkerExit(w, completedAbruptly);
    }
}
```

**这里在获取到并执行任务前加锁，是为了避免在任务运行期间，其他线程调用了shutdown后正在执行的任务被中断，所以为了执行期间运行下去，不响应中断。**

代码6执行清理工作，即将自己这个worker删除，并把完成的任务数量加到线程池的总数量上，代码如下：

```java
private void processWorkerExit(Worker w, boolean completedAbruptly) {
    if (completedAbruptly) // 如果某一个worker在执行while的时候突然中断，则worker数减1
        decrementWorkerCount();

		//加全局锁保证原子更新（删除）workers
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
      //1. 统计线程池完成任务个数，加到全局计数器中，并删除当前这个worker
        completedTaskCount += w.completedTasks;		
        workers.remove(w);
    } finally {
        mainLock.unlock();
    }
    //2. 尝试设置线程池状态为TERMINATED，如果当前是SHUTDOWN状态且工作队列为空
    //或者当前是STOP状态，且线程池里没有活动线程
    tryTerminate();

		//3.如果当前线程个数小于核心个数，则增加
    int c = ctl.get();
    if (runStateLessThan(c, STOP)) {
        if (!completedAbruptly) {
            int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
            if (min == 0 && ! workQueue.isEmpty())
                min = 1;
            if (workerCountOf(c) >= min)
                return; // 正在工作的线程数是大于corlPoolSize，则无需添加新线程，直接返回
        }
        addWorker(null, false);
    }
}
```

存在这个清理工作时因为当前这个worker（线程）获取不到task了，那么它就认为没有任务了，所以会做一些统计工作，并尝试改变线程池的状态。当然，设置线程池状态为TERMINATED后，则还需要调用条件变量termination的signalAll()方法激活所有因为调用线程池的awaitTermination方法而被阻塞的线程。

#### 2.5 shutdown操作

调用shutdown方法后，线程池就不会再接受新的任务了，但是工作队列里面的任务还是要执行。该方法会立刻返回，并不等待队列任务完成再返回。以下是其代码：

```java
public void shutdown() {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        checkShutdownAccess();		//1. 权限检查
        advanceRunState(SHUTDOWN);		//2. 设置当前线程状态为shutdown
        interruptIdleWorkers();			//3. 设置中断标志
        onShutdown(); // hook for ScheduledThreadPoolExecutor
    } finally {
        mainLock.unlock();
    }
    //4.尝试将线程池状态改变为TERMINATED
    tryTerminate();
}
```

其中主要是代码3部分，做了以下事：设置所有空闲线程的中断标志。首先加了全局锁，同时只能有一个线程可以调用shutdown方法。然后尝试获取所有Wokers中自己的锁，获取成功则设置中断标志。若正在执行任务的worker已经获取了自己的锁，则不会被中断。这里中断的是阻塞到getTask()方法并试图从队列里面获取任务的线程，也就是**空闲线程**。代码如下：

```java
private void interruptIdleWorkers(boolean onlyOne) {
    final ReentrantLock mainLock = this.mainLock;		//先加全局锁，保证只有一个线程能中断线程池
    mainLock.lock();
    try {
        for (Worker w : workers) {
            Thread t = w.thread;
            //没有中断且不在执行任务（能获取锁说明没在执行）
            if (!t.isInterrupted() && w.tryLock()) {		
                try {
                    t.interrupt();			//设置中断标志
                } catch (SecurityException ignore) {
                } finally {
                    w.unlock();
                }
            }
            //它为true，则最多只会中断一个worker，甚至有可能没有中断任何worker，不能保证上面的if一定执行
            if (onlyOne)	
                break;
        }
    } finally {
        mainLock.unlock();
    }
}

final void tryTerminate() {
    for (;;) {
        int c = ctl.get();
        ...

        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {		//设置当前线程池状态为TIDYING
                try {
                    terminated();	//默认什么都不做，可能子类会用到
                } finally {
                    ctl.set(ctlOf(TERMINATED, 0));		//设置当前线程状态为TERMINATED
                    termination.signalAll();	//激活因调用条件变量termination的await系列方法而被阻塞的所有线程
                }
                return;
            }
        } finally {
            mainLock.unlock();
        }
        // 不断的进行cas重试上述过程，知道成功
    }
}
```

在如上代码中，首先使用CAS设置当前线程池状态为TIDYING，如果设置成功则执行扩展接口terminated在线程池状态变为TERMINATED前做一些事情，然后设置当前线程池状态为TERMINATED。最后调用termination.signalALL()激活因条件变量termination的await系列方法而被阻塞的所有线程。后面awaitTermination方法会讲到。

#### 2.6 shutdownNow操作

调用shutdownNow方法后，线程池就不会再接受新的任务了，且会丢弃工作队列里面的任务，正在执行的任务会被中断，该方法会立刻返回，并不等待激活的任务执行完成。返回值为这时候队列里面被丢弃的任务列表。

```java
  public List<Runnable> shutdownNow() {
      List<Runnable> tasks;
      final ReentrantLock mainLock = this.mainLock;
      mainLock.lock();
      try {
          checkShutdownAccess();		//1. 检查权限
          advanceRunState(STOP);		//2. 设置线程池状态为STOP
          interruptWorkers();			//3. 中断所有线程，这一步是主要操作
          tasks = drainQueue();		//4.将当前任务队列里的任务移动到tasks列表
      } finally {
          mainLock.unlock();
      }
      tryTerminate();
      return tasks;
  }

  private void interruptWorkers() {
      final ReentrantLock mainLock = this.mainLock;
      mainLock.lock();
      try {
          for (Worker w : workers)
              w.interruptIfStarted();	
      } finally {
          mainLock.unlock();
      }
  }

private final class Worker
  extends AbstractQueuedSynchronizer
  implements Runnable {
    ...
      void interruptIfStarted() {
          Thread t;
          if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
              try {
                  t.interrupt();
              } catch (SecurityException ignore) {
              }
          }
      }
  }
```

要知道代码3会中断的所有线程包括空闲线程和正在执行任务的线程。

#### 2.7 awaitTermination操作

当线程调用awaitTermination方法后，当前线程会被阻塞，直到线程池状态变为TERMINATED才返回，或者等待时间超时才返回。代码如下：

```java
public boolean awaitTermination(long timeout, TimeUnit unit)
    throws InterruptedException {
    long nanos = unit.toNanos(timeout);
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();			//进行判断的时候要加上独占锁
    try {
        for (;;) {
            if (runStateAtLeast(ctl.get(), TERMINATED))		//1. 变为了TERMINATED，返回true
                return true;
            if (nanos <= 0)			//2. 超时了，返回
                return false;
            nanos = termination.awaitNanos(nanos);			//3. 否则说明还有线程在执行等待unit秒
        }
    } finally {
        mainLock.unlock();
    }
}
```