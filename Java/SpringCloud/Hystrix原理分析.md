## 工作流程

### 1. 创建HystrixCommand或HystrixObservableCommand对象

首先构建一个`HystrixCommand`或`HystrixObservableCommand`对象，用来表示对依赖服务的操作请求，同时传递所有需要的参数。从命名方式能看出它使用了“命令模式”实现对服务调用操作的封装。而这两个`Command`对象分别针对不同的应用场景。

- HystrixCommand：用在依赖的服务返回**单个**操作结果的时候。
- HystrixObservableCommand：用在依赖的服务返回**多个**操作结果的时候。

### 2. 命令执行

一共存在4种命令的执行方式，`Hystrix`在执行时会根据创建的`Command`对象以及具体的情况来选择一个执行。

其中`HystrixCommand`实现了下面两个执行方式：

- execute()：同步执行，从依赖的服务返回一个单一的结果对象，或是在发生错误时抛出异常
- queue()：异步执行，直接返回一个`Future`对象，其中包含了服务执行结束时要返回的单一结果对象。

```java
R value = command.execute();
Future<R> fValue = command.queue();
```

而`HystrixObservableCommand`实现了另外两种执行方式：

- observe()：返回`Observable`对象，代表了操作的多个结果，它是一个`Hot Observable`。
- toObservable()：同样返回`Observable`对象，也代表了操作的多个结果，但它返回的是一个`Cold Observable`

```java
Observable<R> ohValue = command.observe();
Observable<R> ocValue = command.toObservable();
```

在`Hystrix`中底层大量使用了RxJava，后面附录介绍了详细情况。这里我们对于事件源`observable`提到了两个不同的概念：`Hot Observable`和`Cold Observable`。其中

- `Hot Observable`是不论“事件源”是否有“订阅者”，都会在创建后对事件进行发布，所以对于`Hot Observable`的每个“订阅者”来说都有可能是从“事件源”的中途开始的，并可能只是看到了整个操作的局部过程。
- `Cold Observable`在没有“订阅者”的时候并不会发布事件，而是进行等待，直到有“订阅者”之后才发布事件，所以对于`Cold Observable`的订阅者，它可以保证从一开始看到整个操作的全部过程。

要注意上面两种Command都使用了RxJava实现，特别是`HystrixCommand`，例如

```java
public R execute() {
		try {
				return queue().get();  //通过queue()返回的一步对象Future<R>的get()方法来实现同步执行
		} catch (Exception e) {
				throw decomposeException(p);
		}
}

public Future<R> queue() {
  	final Observable<R> o = toObservable();
  	final Future<R> f = o.toBlocking().toFuture();
  
  	if (f.isDone()) {
      	//处理立刻抛出的错误
    }
  	return f;
}
```

### 3.结果是否被缓存

若当前命令的请求缓存功能是被启用的，并且该命令缓存命中，那么缓存的结果会立即以`Observable`对象的形式返回。

### 4. 断路器是否打开

命令结果没有缓存命中时，`Hystrix`在执行命令前需要检查断路器是否为打开状态：

- 若断路器打开，那么`Hystrix`不会执行命令，而是转接到`fallback()`处理逻辑，对应于第8步
- 若断路器关闭，那么`Hystrix`跳转到第5步，检查是否有可用资源来执行命令

### 5.线程池/请求队列/信号量是否占满

若与命令相关的线程池和请求队列，或者信号量（不使用线程池时）已经备战满，那么`Hystrix`也不会执行命令，而是转到`fallback()`处理逻辑（第8步）。

要注意这里的线程池不是容器的线程池，而是**每个依赖服务的专有线程池**。`Hystrix`为了保证不会因为某个依赖服务的问题影响到其他的依赖服务而采用了“舱壁模式”来隔离每个依赖的服务。

### 6.HystrixObservableCommand.construct()或HystrixCommand.run()

`Hystrix`会根据我们编写的方法来决定采取什么样的方式去请求依赖服务：

- `HystrixCommand.run()`：返回一个单一的结果，或者抛出异常
- `HystrixObservableCommand.construct()`：返回一个`Observabe`对象来反射多个结果，或通过`onError()`发送错误通知。











## 附录

### 命令模式

命令模式，将来自客户端的请求封装成一个对象，从而让你可使用不同的请求对客户端进行参数化。它可以被用于实现“行为请求者”与“请求实现者”的解耦，以便使两者可以适应变化。可以看一个例子：



### RxJava 观察者-订阅者模式

`Observable`对象可理解为“事件源”或“被观察者”，与之对应的`Subscriber`对象，可理解为“订阅者”或是“观察者”。

- `Observable`用来向订阅者`Subscriber`对象分布事件，`Subscriber`对象则在接收到事件后对其进行处理。
- 一个`Observable`可发出多个事件，直到结束或是发出异常。
- `Observable`对象每发出一个事件，**就会调用对应观察者`Subscriber`对象的`onNext()`方法**。
- 每个`Observable`的执行，最后一定会通过调用`Subscriber.onCompleted()`或者`Subscriber.onError()`来结束该事件的操作流。

可通过下面这个例子来直观理解一下`Observable`与`Subscriber`。

```java
public class ObserverTest {

    public static void main(String[] args) {
        //事件源
        Observable<String> observable = Observable.create(new Observable.OnSubscribe<String>() {
            @Override
            public void call(Subscriber<? super String> subscriber) {
                subscriber.onNext("Hello RxJava");
                subscriber.onNext("I am 程序员 DD");
                subscriber.onCompleted();
            }
        });

        Subscriber<String> subscriber = new Subscriber<String>() {
            @Override
            public void onCompleted() {
            }

            @Override
            public void onError(Throwable throwable) {
            }

            @Override
            public void onNext(String s) {
                System.out.println("Subscriber ：" + s);
            }
        };
        //通过subscribe来触发事件的发布
        observable.subscribe(subscriber);
    }
}
```

注意当执行`observable.subscribe(subscriber)`来触发事件的发布。