## 什么是Coroutine？

Coroutine，又称作**协程**。从字面上来理解，即协同运行的例程，它是比是线程（thread）更细量级的**用户态线程**，特点是允许用户的主动调用和主动退出，挂起当前的例程然后返回值或去执行其他任务，接着返回原来停下的点继续执行，使用的是yield语句。其实这里我们要感谢操作系统（OS）为我们做的工作，因为它具有getcontext和swapcontext这些特性，通过系统调用，我们可以把上下文和状态保存起来，切换到其他的上下文，这些特性为coroutine的实现提供了底层的基础。操作系统的Interrupts（异常）和Traps（陷入）机制则为这种实现提供了可能性，因此它看起来可能是下面这样的:

![系统调用](https://user-gold-cdn.xitu.io/2016/11/29/5d462181b9290027b38fb51048e42750?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

## 理解生成器（generator）

学过生成器和迭代器的同学应该都知道python有`yield`这个关键字，yield能把一个函数变成一个generator，与return不同，**yield在函数中返回值时会保存函数的状态，使下一次调用函数时会从上一次的状态继续执行，即从yield的下一条语句开始执行**，这样做有许多好处，比如我们想要生成一个数列，若该数列的存储空间太大，而我们仅仅需要访问前面几个元素，那么yield就派上用场了，它实现了这种一边循环一边计算的机制，节省了存储空间，提高了运行效率。

这里以斐波那契数列为例：

```python
def fib(max):
    n, a, b = 0, 0, 1
    while n  max:
        print b
        a, b = b, a + b
        n = n + 1
```

如果使用上述的算法，那么我每一次调用函数时，都要耗费大量时间循环做重复的事情。而如果使用yield的话，它则会生成一个generator，当我需要时，调用它的next方法获得下一个值，改动的方法很简单，直接把print改为yield就OK。

## 生产者－消费者的协程

下面这个例子是典型的生产者－消费者问题，我们用协程的方式来实现它。首先从主程序中开始看，第一句`c = consumer()`，因为consumer函数中存在yield语句，python会把它当成一个generator（生成器，注意：生成器和协程的概念区别很大，等会再说，千万别混淆了两者），因此在运行这条语句后，python并不会像执行函数一样，**而是返回了一个generator object**。

再看第二条语句`c.send(None)`，这条语句的作用是将consumer（即变量c，它是一个generator）中的语句推进到第一个yield语句出现的位置，那么在例子中，consumer中的`status = True`和`while True:`都已经被执行了，程序停留在`n = yield status`的位置（注意：此时这条语句还没有被执行），上面说的send(None)语句十分重要，如果漏写这一句，那么程序直接报错，这个send()方法看上去似乎挺神奇，等下再讲他的作用。

下面第三句`p = producer(c)`，这里则像上面一样**定义了producer的生成器**，注意的是这里我们传入了消费者的生成器，来让producer跟consumer通信。

第四句`for status in p:`，这条语句会循环地运行producer和获取它yield回来的状态。

好了，进入正题，**在我们要让生产者发送1,2,3,4,5给消费者，消费者接受数字，返回状态给生产者，而我们的消费者只需要3,4,5就行了，当数字等于3时，会返回一个错误的状态。最终我们需要由主程序来监控生产者－消费者的过程状态，调度结束程序。**

现在程序流进入了producer里面，我们直接看`yield consumer.send(n)`，生产者调用了消费者的send()方法，把n发送给consumer（即c），在consumer中的`n = yield status`，**n拿到的是消费者发送的数字**，同时，consumer用yield的方式把状态（status）返回给消费者，注意：这时producer（即消费者）的`consumer.send()`调用返回的就是consumer中yield的status！消费者马上将status返回给调度它的主程序，主程序获取状态，判断是否错误，若错误，则终止循环，结束程序。上面看起来有点绕，其实这里面`generator.send(n)`的作用是：**把n发送generator(生成器)中yield的赋值语句中，同时返回generator中yield的变量（结果）**。

于是程序便一直运作，直至consumer中获取的n的值变为3！此时consumer把status变为False，最后返回到主程序，主程序中断循环，程序结束。（观察输出结果，是否如你所想？）

这种协程的方式保证了程序以**有序地方式协作运行例程**，而且它还能保存函数中的上下文状态（看看status），使每次重新进入时都能获取以前上下文的变量值。

```python
#_*_ coding:utf-8

def consumer():
    status = True
    while status:
        n = yield status
        print "我拿到了{}".format(n)
        if n == 3:
            status = False

def producer(consumer):
    n = 5
    while n > 0:
        # yield给主程序返回消费者的状态
        yield consumer.send(n)
        n -= 1

if __name__ == '__main__':
    c = consumer()
    c.send(None)
    p = producer(c)
    for status in p:
        if status == False:
            print "我只要3,4,5就行啦"
            break
    print "程序结束"
```

## Coroutine与Generator

有些人会把生成器（generator）和协程（coroutine）的概念混淆，我以前也会这样，不过其实发现，两者的区别还是很大的。

直接上最重要的区别：

- generator总是生成值，一般是迭代的序列
- coroutine关注的是消耗值，是数据(data)的消费者
- coroutine不会与迭代操作关联，而generator会
- coroutine强调协同控制程序流，generator强调保存状态和产生数据

相似的是，它们都是不用return来实现重复调用的函数/对象，都用到了yield(中断/恢复)的方式来实现。

## 管道

下面展示如何用coroutine实现管道。

![管道](https://lc-api-gold-cdn.xitu.io/2c165601b558b163f2c1?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

首先了解管道的要素：

- 要有起始的源（source）

![起始的源](https://user-gold-cdn.xitu.io/2016/11/29/0b1289f95ffe68c0235174e9e07d93db?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

- 要有终点（end-point）

![终点](https://lc-api-gold-cdn.xitu.io/ea26cbab4747140cd297?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

首先，我们在source建立一个模拟Unix的`tail -f`命令的coroutine。

```python
import time
def follow(thefile, target):
    thefile.seek(0,0)      # 到达文件的开始，具体查看seek函数及参数
    while True:
         line = thefile.readline()
         if not line:
             time.sleep(0.1)
             continue
         coun = target.send(line)  # 将读取的消息发送给target，并接收返回的count
```

它接受一个文件以及一个coroutine对象作为参数，用send()把文件每一行传递到管道一侧的coroutine中。

现在，我们再为管道接上终点（end-point），也就是下面的printer()，它将负责打印行结果。
这里定义了coroutine作为装饰器，帮我们做了send(None)的操作，免去了我们的手动调用。

```python
def coroutine(func):
    def wrapper(*args, **kws):
        cr = func(*args, **kws)
        cr.send(None)    # 每次打印完成后，仍然开启这个协程
        return cr
    return wrapper

@coroutine
def printer():
    count = 1
    while True:
        # 协程首先运行到这里，等待send信号，并接收对应的参数
        # 当yield后面没有参数时，表示不返回数据
        # 否则会返回yield后面的数据，这里我们返回了一个count
        line = yield count
        print line
        count += 1
```

注意了，目前为止，我们的管道是这样的：follow -> printer。

下面我们尝试使用它，跟踪读取一个日志文件：

```python
if __name__ == '__main__':
    f = open('readme.txt')
    follow(f, printer())
```

而现在，我们在管道中间定义一个过滤器，同样的，模拟的是UNIX中grep的功能。

```python
import time

def follow(thefile, target):
    thefile.seek(0, 0)  # 到达文件底部
    while True:
        line = thefile.readline()
        if not line:  # 读到末尾了
            time.sleep(0.1)
            continue
        target.send(line)  

# 装饰器
def coroutine(func):
    def wrapper(*args, **kws):
        cr = func(*args, **kws)
        cr.send(None)    # 每次yield返回后，重新开启这个协程
        return cr
    return wrapper

@coroutine
def printer():
    while True:
        line = (yield)  # 接受到行，并打印
        print line

@coroutine
def grep(pattern, target):
    print 'Looking for %s' % pattern
    while True:
        line = (yield)  # 先接受到从follow传过来的行
        if pattern in line:
            print 'I found the pattern in %s' % line
        target.send(line)    # 再把接受到的行发给下一个，即printer


if __name__ == '__main__':
    f = open('readme.txt')
    follow(f, grep('python', printer()))
```

## 事件处理

用coroutine使事件处理的方式更加简单。试想一下Python中的XML Parser，里面就有Event Handling的概念。转换成coroutine way，我们可以借助send()方法，接受和处理Handler中发来的数据。

![事件分发](https://user-gold-cdn.xitu.io/2016/11/29/bf35792747a4afdd045c706e9851b939?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

下面演示了一个爬取公交车信息的例子，它使用了xml.sax来解析读取的xml信息，其中用到了事件处理和数据过滤。