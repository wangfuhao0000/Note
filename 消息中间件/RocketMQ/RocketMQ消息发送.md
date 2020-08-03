RocketMQ发送普通消息有三种实现方式：

- 可靠同步发送：发送者向MQ执行发送消息API时，同步等待，直到服务器返回发送结果
- 可靠异步发送：执行发送消息API时，可指定消息发送成功后的回调函数，调用后立即返回，消息发送成功或失败的回调任务在一个新线程中执行。
- 单向发送：执行发送消息API时，直接返回，不等待消息服务器的结果，也不注册回调函数。

##  认识RocketMQ消息

封装类是org.apache.rocketmq.common.message.Message。主要包含的属性如下：

- String topic：消息所属主题
- int flag：消息Flag，RocketMQ对此标志不做处理
- Map properties：消息的扩展属性，因为可能会有额外的属性，所以存储在此Map中
- byte[] body：消息的主体内容

其中扩展属性properties会包含我们常用的一些如下：

- tag：消息TAG，用于消息过滤
- keys：**Message索引键**，多个用空格隔开，RocketMQ可根据这些key快速检索到消息
- waitStoreMsgOK：消息发送时是否等消息存储完成后再返回
- delayTimeLevel：消息延迟级别，用于定时消息或消息重试。

## 生产者启动流程

生产者在Client模块中。所以我们先看看消息发送者是啥样的。

### DefaultMQProducer消息发送者

先看看这个类的一些主要方法：

1. ```java
   void createTopic(String key, String newTopic, int queueNum, int topicSysFlag)
   ```



### 消息生产者启动流程

我们从DefaultMQProducerImpl#start方法开始看起：

```java
public void start() throws MQClientException {
    this.setProducerGroup(withNamespace(this.producerGroup));
    this.defaultMQProducerImpl.start();
    if (null != traceDispatcher) {
        try {
            traceDispatcher.start(this.getNamesrvAddr(), this.getAccessChannel());
        } catch (MQClientException e) {
            log.warn("trace dispatcher start failed ", e);
        }
    }
}
```