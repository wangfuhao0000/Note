一、Thread线程类API

1.4.3interrupt方法

线程中断在之前的版本有stop方法，但是被设置过时了。现在已经**没有强制线程终止**的方法了！

由于stop方法可以让**一个线程A终止掉另一个线程B**

- 被终止的线程B会立即释放锁，这可能会让**对象处于不一致的状态**。
- **线程A也不知道线程B什么时候能够被终止掉**，万一线程B还处理运行计算阶段，线程A调用stop方法将线程B终止，那就很无辜了~

总而言之，Stop方法太暴力了，不安全，所以被设置过时了。

我们一般使用的是interrupt来**请求终止线程**~

- 要注意的是：interrupt**不会真正停止**一个线程，它仅仅是给这个线程发了一个信号告诉它，它应该要结束了(明白这一点非常重要！)
- 也就是说：Java设计者实际上是**想线程自己来终止**，通过上面的**信号**，就可以判断处理什么业务了。
- 具体到底中断还是继续运行，应该**由被通知的线程自己处理**

```
Thread t1 = new Thread( new Runnable(){
    public void run(){
        // 若未发生中断，就正常执行任务
        while(!Thread.currentThread.isInterrupted()){
            // 正常任务代码……
        }
        // 中断的处理代码……
        doSomething();
    }
} ).start();
```

再次说明：调用interrupt()**并不是要真正终止掉当前线程**，仅仅是设置了一个中断标志。这个中断标志可以给我们用来判断**什么时候该干什么活**！什么时候中断**由我们自己来决定**，这样就可以**安全地终止线程**了！

我们来看看源码是怎么讲的吧：

![img](https://note.youdao.com/yws/public/resource/56347de0c9d0b624bcbb02366ca7a2b8/xmlnote/WEB042952bbcccd5fc1275e8ac49977f3c3/CE61D79CA2CF47B4B24F4B9391B35A1C/16356)

再来看看刚才说抛出的异常是什么东东吧：

![img](https://note.youdao.com/yws/public/resource/56347de0c9d0b624bcbb02366ca7a2b8/xmlnote/WEB042952bbcccd5fc1275e8ac49977f3c3/36A0BFDEA22D4FDC81757D3AC175FDE6/16373)

所以说：**interrupt方法压根是不会对线程的状态造成影响的，它仅仅设置一个标志位罢了**

interrupt线程中断还有另外**两个方法(检查该线程是否被中断)**：

- 静态方法interrupted()-->**会清除中断标志位**
- 实例方法isInterrupted()-->**不会清除中断标志位**

![img](https://note.youdao.com/yws/public/resource/56347de0c9d0b624bcbb02366ca7a2b8/xmlnote/WEB042952bbcccd5fc1275e8ac49977f3c3/E94177946C2D4BEE88502F54F3A81EE1/16348)

![img](https://note.youdao.com/yws/public/resource/56347de0c9d0b624bcbb02366ca7a2b8/xmlnote/WEB042952bbcccd5fc1275e8ac49977f3c3/49385EBD460D42F3AE667DC68B8E1CB8/16351)

上面还提到了，如果阻塞线程调用了interrupt()方法，那么会**抛出异常，设置标志位为false，同时该线程会退出阻塞**的。我们来测试一波：

```
public class Main {
    /**
     * @param args
     */
    public static void main(String[] args) {
        Main main = new Main();

        // 创建线程并启动
        Thread t = new Thread(main.runnable);
        System.out.println("This is main ");
        t.start();

        try {

            // 在 main线程睡个3秒钟
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            System.out.println("In main");
            e.printStackTrace();
        }

        // 设置中断
        t.interrupt();
    }

    Runnable runnable = () -> {
        int i = 0;
        try {
            while (i < 1000) {

                // 睡个半秒钟我们再执行
                Thread.sleep(500);

                System.out.println(i++);
            }
        } catch (InterruptedException e) {


            // 判断该阻塞线程是否还在
            System.out.println(Thread.currentThread().isAlive());

            // 判断该线程的中断标志位状态
            System.out.println(Thread.currentThread().isInterrupted());

            System.out.println("In Runnable");
            e.printStackTrace();
        }
    };
}
```

结果：

![img](https://note.youdao.com/yws/public/resource/56347de0c9d0b624bcbb02366ca7a2b8/xmlnote/WEB042952bbcccd5fc1275e8ac49977f3c3/0D29785E03B447B9B55D93064B87F472/16350)

接下来我们分析它的**执行流程**是怎么样的：

![img](https://note.youdao.com/yws/public/resource/56347de0c9d0b624bcbb02366ca7a2b8/xmlnote/WEB042952bbcccd5fc1275e8ac49977f3c3/F1B784374BE849BDAB648227671257D8/16372)