想象以下情形：长途客车在路途上，有人上车有人下车，但是乘客总是希望能够在客车上得到休息。
传统的做法是：每隔一段时间（或每一个站），司机或售票员对每一个乘客询问是否下车。
 Reactor模式做法是：汽车是乘客访问的主体（Reactor），乘客上车后，到售票员（acceptor）处登记，之后乘客便可以休息睡觉去了，当到达乘客所要到达的目的地后，售票员将其唤醒即可。

## 概念

Reactor模式是基于事件驱动的分发处理模型。有一个或多个并发输入源，有一个**Service Handler**，有多个**Request Handlers**。这个Service Handler会同步的将输入的请求（Event）多路复用的分发给相应的Request Handler。如果用图来表达：

![img](https:////upload-images.jianshu.io/upload_images/5682416-7a989dc89fea40a7.png?imageMogr2/auto-orient/strip|imageView2/2/w/740/format/webp)

## 思想：分而治之+事件驱动

1）分而治之
 一个连接里完整的网络处理过程一般分为accept、read、decode、process、encode、send这几步。Reactor模式将每个步骤映射为一个Task，**服务端线程执行的最小逻辑单元不再是一次完整的网络请求，而是Task**，且采用非阻塞方式执行。

2）事件驱动
每个Task对应特定网络事件。**当Task准备就绪时，Reactor收到对应的网络事件通知，并将Task分发给绑定了对应网络事件的Handler执行。**

3）几个角色
 reactor：负责绑定管理事件和处理接口；
 selector：负责监听响应事件，将事件分发给绑定了该事件的Handler处理；
 Handler：事件处理器，绑定了某类事件，负责执行对应事件的Task对事件进行处理；
 Acceptor：Handler的一种，绑定了connect事件。当客户端发起connect请求时，Reactor会将accept事件分发给Acceptor处理。

## Reactor模式结构

![img](https:////upload-images.jianshu.io/upload_images/5682416-dd2ca2209c1b7209.png?imageMogr2/auto-orient/strip|imageView2/2/w/479/format/webp)

- Handle 句柄；用来标识socket连接或是打开文件；
- Synchronous Event Demultiplexer：同步事件多路分解器：由操作系统内核实现的一个函数；用于阻塞等待发生在句柄集合上的一个或多个事件；（如select/epoll；）
- Event Handler：事件处理接口
- Concrete Event HandlerA：实现应用程序所提供的特定事件处理逻辑；
- Reactor：反应器，定义一个接口，实现以下功能：
   1）供应用程序注册和删除关注的事件句柄；
   2）运行事件循环；
   3）有就绪事件到来时，分发事件到之前注册的回调函数上处理；

> “反应”即“倒置”，“控制逆转”

## 流程

![img](https:////upload-images.jianshu.io/upload_images/5682416-7d63ea514fed905a.png?imageMogr2/auto-orient/strip|imageView2/2/w/666/format/webp)

## 举例

### 登录——Accept新链接

![img](https:////upload-images.jianshu.io/upload_images/5682416-1d045fe589b345ea.png?imageMogr2/auto-orient/strip|imageView2/2/w/718/format/webp)

### 新连接Read和Write过程

![img](https:////upload-images.jianshu.io/upload_images/5682416-cc08766d5c643224.png?imageMogr2/auto-orient/strip|imageView2/2/w/690/format/webp)

## 实现

### 单线程Reactor

![img](https:////upload-images.jianshu.io/upload_images/5682416-75f2862a430f6261.png?imageMogr2/auto-orient/strip|imageView2/2/w/662/format/webp)



1）优点：
 不需要做并发控制，代码实现简单清晰。

2）缺点：

a)不能利用多核CPU；
 b)一个线程需要执行处理所有的accept、read、decode、process、encode、send事件，处理成百上千的链路时性能上无法支撑；
 c)一旦reactor线程意外跑飞或者进入死循环，会导致整个系统通信模块不可用。

### 多线程Reactor

![img](https:////upload-images.jianshu.io/upload_images/5682416-acd02c0cbbb84d9e.png?imageMogr2/auto-orient/strip|imageView2/2/w/666/format/webp)

特点：

a)有专门一个reactor线程用于监听服务端ServerSocketChannel，接收客户端的TCP连接请求；
 b)网络IO的读/写操作等由一个worker reactor线程池负责，由线程池中的NIO线程负责监听SocketChannel事件，进行消息的读取、解码、编码和发送。
 c)一个NIO线程可以同时处理N条链路，但是一个链路只注册在一个NIO线程上处理，防止发生并发操作问题。

### 主从多线程

![img](https:////upload-images.jianshu.io/upload_images/5682416-9d2613f3b3d8211c.png?imageMogr2/auto-orient/strip|imageView2/2/w/674/format/webp)



在绝大多数场景下，Reactor多线程模型都可以满足性能需求；但是在极个别特殊场景中，一个NIO线程负责监听和处理所有的客户端连接可能会存在性能问题。

特点：

a)服务端用于接收客户端连接的不再是个1个单独的reactor线程，而是一个boss reactor线程池；

b)服务端启用多个ServerSocketChannel监听不同端口时，每个ServerSocketChannel的监听工作可以由线程池中的一个NIO线程完成。