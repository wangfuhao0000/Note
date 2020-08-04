## 1. 前言

消息队列的性能好坏，其文件存储机制设计是衡量一个消息队列服务技术水平和最关键指标之一。下面将从Kafka文件**存储机制**和**物理结构**角度，分析Kafka是如何实现高效文件存储，及实际应用效果。

### 1.1  Kafka的特性：

- 高吞吐量、低延迟：kafka每秒可以处理几十万条消息，它的延迟最低只有几毫秒，每个topic可以分多个partition，**consumer group 对partition进行consume操作**。

- 可扩展性：kafka集群支持热扩展

- 持久性、可靠性：消息被持久化到本地磁盘，并且支持数据备份防止数据丢失

- 容错性：允许集群中节点失败（若副本数量为n,则允许n-1个节点失败）

- 高并发：支持数千个客户端同时读写

### 1.2  Kafka的使用场景：

- 日志收集：一个公司可以用Kafka可以收集各种服务的log，通过kafka以统一接口服务的方式开放给各种consumer，例如hadoop、Hbase、Solr等。

- 消息系统：解耦和生产者和消费者、缓存消息等。

- 用户活动跟踪：Kafka经常被用来记录web用户或者app用户的各种活动，如浏览网页、搜索、点击等活动，这些活动信息被各个服务器发布到kafka的topic中，然后订阅者通过订阅这些topic来做实时的监控分析，或者装载到hadoop、数据仓库中做离线分析和挖掘。

- 运营指标：Kafka也经常用来记录运营监控数据。包括收集各种分布式应用的数据，生产各种操作的集中反馈，比如报警和报告。

- 流式处理：比如spark streaming和storm

- 事件源

### 1.3 kafka的设计思想

#### kafka Broker Leader的选举

Kakfa Broker集群受Zookeeper管理。所有的Kafka Broker节点一起去Zookeeper上注册一个**临时节点**，因为只有一个Kafka Broker会注册成功，其他的都会失败，所以这个成功在Zookeeper上注册临时节点的这个Kafka Broker会成为**Kafka Broker Controller**，其他的Kafka broker叫**Kafka Broker follower**。（这个过程叫Controller在ZooKeeper注册Watch）。这个Controller会监听其他的Kafka Broker的所有信息，**如果这个kafka broker controller宕机了，在zookeeper上面的那个临时节点就会消失，此时所有的kafka broker又会一起去Zookeeper上注册一个临时节点**，因为只有一个Kafka Broker会注册成功，其他的都会失败，所以这个成功在Zookeeper上注册临时节点的这个Kafka Broker会成为Kafka Broker Controller，其他的Kafka broker叫Kafka Broker follower。

#### Consumergroup

各个consumer线程可以组成一个组（Consumer group），**partition中的每个message只能被组（Consumer group）中的一个consumer（consumer 线程）消费，如果一个message可以被多个consumer（consumer 线程）消费的话，那么这些consumer必须在不同的组**。Kafka不支持一个partition中的message由两个或两个以上的同一个consumer group下的consumer thread来处理，除非再启动一个新的consumer group。所以如果想同时对一个topic做消费的话，启动多个consumer group就可以了，但是要注意的是，这里的多个consumer的消费都必须是顺序读取partition里面的message，新启动的consumer默认从partition队列最头端最新的地方开始阻塞的读message。它不能像AMQ那样可以多个BET作为consumer去互斥的（for update悲观锁）并发处理message，这是因为多个BET去消费一个Queue中的数据的时候，由于要保证不能多个线程拿同一条message，所以就需要行级别悲观锁（for update）,这就导致了consume的性能下降，吞吐量不够。而kafka为了保证吞吐量，只允许同一个consumer group下的一个consumer线程去访问一个partition。如果觉得效率不高的时候，可以加partition的数量来横向扩展，那么再加新的consumer thread去消费。如果想多个不同的业务都需要这个topic的数据，起多个consumer group就好了，大家都是顺序的读取message，offsite的值互不影响。这样没有锁竞争，充分发挥了横向的扩展性，吞吐量极高。这也就形成了分布式消费的概念。

当启动一个consumer group去消费一个topic的时候，无论topic里面有多个少个partition，无论我们consumer group里面配置了多少个consumer thread，**这个consumer group下面的所有consumer thread一定会消费全部的partition**；即便这个consumer group下只有一个consumer thread，那么这个consumer thread也会去消费所有的partition。因此，最优的设计就是，**consumer group下的consumer thread的数量等于partition数量**，这样效率是最高的。

同一partition的一条message只能被同一个Consumer Group内的一个Consumer消费。不能够一个consumer group的多个consumer同时消费一个partition。

一个consumer group下，无论有多少个consumer，这个consumer group一定回去把这个topic下所有的partition都消费了。当consumer group里面的consumer数量小于这个topic下的partition数量的时候，如下图groupA,groupB，就会出现一个conusmer thread消费多个partition的情况，总之是这个topic下的partition都会被消费。如果consumer group里面的consumer数量等于这个topic下的partition数量的时候，如下图groupC，此时效率是最高的，每个partition都有一个consumer thread去消费。当consumer group里面的consumer数量大于这个topic下的partition数量的时候，如下图GroupD，就会有一个consumer thread空闲。因此，我们在设定consumer group的时候，只需要指明里面有几个consumer数量即可，无需指定对应的消费partition序号，consumer会自动进行rebalance。

多个Consumer Group下的consumer可以消费同一条message，但是这种消费也是以o（1）的方式顺序的读取message去消费,，所以一定会重复消费这批message的，不能向AMQ那样多个BET作为consumer消费（对message加锁，消费的时候不能重复消费message）

#### Consumer Rebalance的触发条件

1. Consumer增加或删除会触发 Consumer Group的Rebalance
2. Broker的增加或者减少都会触发 Consumer Rebalance

#### Consumer

Consumer处理partition里面的message的时候是o（1）顺序读取的。所以必须维护着上一次读到哪里的offsite信息。high level API,offset存于Zookeeper中，low level API的offset由自己维护。一般来说都是使用high level api的。Consumer的delivery gurarantee，默认是读完message先commmit再处理message，autocommit默认是true，这时候先commit就会更新offsite+1，一旦处理失败，offsite已经+1，这个时候就会丢message；也可以配置成读完消息处理再commit，这种情况下consumer端的响应就会比较慢的，需要等处理完才行。

一般情况下，一定是一个consumer group处理一个topic的message。Best Practice是这个consumer group里面consumer的数量等于topic里面partition的数量，这样效率是最高的，一个consumer thread处理一个partition。如果这个consumer group里面consumer的数量小于topic里面partition的数量，就会有consumer thread同时处理多个partition（这个是kafka自动的机制，我们不用指定），但是总之这个topic里面的所有partition都会被处理到的。。如果这个consumer group里面consumer的数量大于topic里面partition的数量，多出的consumer thread就会闲着啥也不干，剩下的是一个consumer thread处理一个partition，这就造成了资源的浪费，因为一个partition不可能被两个consumer thread去处理。所以我们线上的分布式多个service服务，每个service里面的kafka consumer数量都小于对应的topic的partition数量，但是所有服务的consumer数量只和等于partition的数量，这是因为分布式service服务的所有consumer都来自一个consumer group，如果来自不同的consumer group就会处理重复的message了（同一个consumer group下的consumer不能处理同一个partition，不同的consumer group可以处理同一个topic，那么都是顺序处理message，一定会处理重复的。一般这种情况都是两个不同的业务逻辑，才会启动两个consumer group来处理一个topic）。

**如果producer的流量增大，当前的topic的parition数量=consumer数量，这时候的应对方式就是很想扩展：增加topic下的partition，同时增加这个consumer group下的consumer。

![img](https://upload-images.jianshu.io/upload_images/8542343-573c15e8b7811933?imageMogr2/auto-orient/strip|imageView2/2/w/999/format/webp)

#### Delivery Mode

Kafka producer 发送message不用维护message的offsite信息，因为这个时候，offsite就相当于一个自增id，producer就尽管发送message就好了。而且Kafka与AMQ不同，AMQ大都用在处理业务逻辑上，而Kafka大都是日志，所以Kafka的producer一般都是大批量的batch发送message，向这个topic一次性发送一大批message，load balance到一个partition上，一起插进去，offsite作为自增id自己增加就好。但是Consumer端是需要维护这个partition当前消费到哪个message的offsite信息的，这个offsite信息，high level api是维护在Zookeeper上，low level api是自己的程序维护。（Kafka管理界面上只能显示high level api的consumer部分，因为low level api的partition offsite信息是程序自己维护，kafka是不知道的，无法在管理界面上展示 ）当使用high level api的时候，先拿message处理，再定时自动commit offsite+1（也可以改成手动）, 并且kakfa处理message是没有锁操作的。因此如果处理message失败，此时还没有commit offsite+1，当consumer thread重启后会重复消费这个message。但是作为高吞吐量高并发的实时处理系统，at least once的情况下，至少一次会被处理到，是可以容忍的。如果无法容忍，就得使用low level api来自己程序维护这个offsite信息，那么想什么时候commit offsite+1就自己搞定了。

-**Topic & Partition：**Topic相当于传统消息系统MQ中的一个队列queue，producer端发送的message必须指定是发送到哪个topic，但是不需要指定topic下的哪个partition，因为kafka会把收到的message进行load balance，均匀的分布在这个topic下的不同的partition上（ hash(message) % [broker数量]  ）。物理上存储上，这个topic会分成一个或多个partition，每个partiton相当于是一个子queue。在物理结构上，每个partition对应一个物理的目录（文件夹），文件夹命名是[topicname]_[partition]_[序号]，一个topic可以有无数多的partition，根据业务需求和数据量来设置。在kafka配置文件中可随时更高num.partitions参数来配置更改topic的partition数量，在创建Topic时通过参数指定parittion数量。Topic创建之后通过Kafka提供的工具也可以修改partiton数量。

一般来说，（1）一个Topic的Partition数量大于等于Broker的数量，可以提高吞吐率。（2）同一个Partition的Replica尽量分散到不同的机器，高可用。

**当add a new partition的时候**，partition里面的message不会重新进行分配，原来的partition里面的message数据不会变，新加的这个partition刚开始是空的，随后进入这个topic的message就会重新参与所有partition的load balance

-**Partition Replica：**每个partition可以在其他的kafka broker节点上存副本，以便某个kafka broker节点宕机不会影响这个kafka集群。存replica副本的方式是按照kafka broker的顺序存。例如有5个kafka broker节点，某个topic有3个partition，每个partition存2个副本，那么partition1存broker1,broker2，partition2存broker2,broker3。。。以此类推**（replica副本数目不能大于kafka broker节点的数目，否则报错。这里的replica数其实就是partition的副本总数，其中包括一个leader，其他的就是copy副本）**。这样如果某个broker宕机，其实整个kafka内数据依然是完整的。但是，replica副本数越高，系统虽然越稳定，但是回来带资源和性能上的下降；replica副本少的话，也会造成系统丢数据的风险。

（1）怎样传送消息：producer先把message发送到partition leader，再由leader发送给其他partition follower。（如果让producer发送给每个replica那就太慢了）

（2）在向Producer发送ACK前需要保证有多少个Replica已经收到该消息：根据ack配的个数而定

（3）怎样处理某个Replica不工作的情况：如果这个部工作的partition replica不在ack列表中，就是producer在发送消息到partition leader上，partition leader向partition follower发送message没有响应而已，这个不会影响整个系统，也不会有什么问题。如果这个不工作的partition replica在ack列表中的话，producer发送的message的时候会等待这个不工作的partition replca写message成功，但是会等到time out，然后返回失败因为某个ack列表中的partition replica没有响应，此时kafka会自动的把这个部工作的partition replica从ack列表中移除，以后的producer发送message的时候就不会有这个ack列表下的这个部工作的partition replica了。

（4）怎样处理Failed Replica恢复回来的情况：如果这个partition replica之前不在ack列表中，那么启动后重新受Zookeeper管理即可，之后producer发送message的时候，partition leader会继续发送message到这个partition follower上。如果这个partition replica之前在ack列表中，此时重启后，需要把这个partition replica再手动加到ack列表中。（ack列表是手动添加的，出现某个部工作的partition replica的时候自动从ack列表中移除的）

## 面试题

### 1. Kafka如何处理丢失消息

生产者(Producer) 调用`send`方法发送消息之后，消息可能因为网络问题并没有发送过去。

所以，我们不能默认在调用`send`方法发送消息之后消息消息发送成功了。为了确定消息是发送成功，我们要判断消息发送的结果。但是要注意的是  Kafka 生产者(Producer) 使用  `send` 方法发送消息实际上是**异步的操作**，我们可以通过 `get()`方法获取调用结果，但是这样也让它变为了同步操作，示例代码如下：

```java
SendResult<String, Object> sendResult = kafkaTemplate.send(topic, o).get();
if (sendResult.getRecordMetadata() != null) {
  logger.info("生产者成功发送消息到" + sendResult.getProducerRecord().topic() + "-> " + sendRe
              sult.getProducerRecord().value().toString());
}
```

但是一般不推荐这么做！可以采用为其**添加回调函数**的形式，示例代码如下：

```java
ListenableFuture<SendResult<String, Object>> future = kafkaTemplate.send(topic, o);
future.addCallback(result -> logger.info("生产者成功发送消息到topic:{} partition:{}的消息", result.getRecordMetadata().topic(), result.getRecordMetadata().partition()),
        ex -> logger.error("生产者发送消失败，原因：{}", ex.getMessage()));
```

如果消息发送失败的话，我们检查失败的原因之后重新发送即可！

> 另外这里推荐为 Producer 的`retries`（**重试次数**）设置一个比较合理的值，一般是 3 ，但是为了保证消息不丢失的话一般会设置比较大一点。设置完成之后，当出现网络问题之后能够自动重试消息发送，避免消息丢失。另外，建议还要设置**重试间隔**，因为间隔太小的话重试的效果就不明显了，网络波动一次你3次一下子就重试完了。

#### 消费者丢失消息的情况

我们知道消息在被追加到 Partition(分区)的时候都会分配一个特定的偏移量（offset）。偏移量（offset)表示 Consumer 当前消费到的 Partition(分区)的所在的位置。Kafka 通过偏移量（offset）可以保证消息在分区内的顺序性。



![kafka offset](https://user-gold-cdn.xitu.io/2020/3/16/170e29d648e63e5d?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



当消费者拉取到了分区的某个消息之后，消费者会自动提交了 offset。自动提交的话会有一个问题，试想一下，当消费者刚拿到这个消息准备进行真正消费的时候，突然挂掉了，消息实际上并没有被消费，但是 offset 却被自动提交了。

**解决办法也比较粗暴，我们手动关闭闭自动提交 offset，每次在真正消费完消息之后之后再自己手动提交 offset 。** 但是，细心的朋友一定会发现，这样会带来消息被重新消费的问题。比如你刚刚消费完消息之后，还没提交 offset，结果自己挂掉了，那么这个消息理论上就会被消费两次。

#### Kafka 弄丢了消息

我们知道 Kafka 为分区（Partition）引入了多副本（Replica）机制。分区（Partition）中的多个副本之间会有一个叫做 leader 的家伙，其他副本称为 follower。我们发送的消息会被发送到 leader 副本，然后 follower 副本才能从 leader 副本中拉取消息进行同步。生产者和消费者只与 leader 副本交互。你可以理解为其他副本只是 leader 副本的拷贝，它们的存在只是为了保证消息存储的安全性。

**试想一种情况：假如 leader 副本所在的 broker 突然挂掉，那么就要从 follower 副本重新选出一个 leader ，但是 leader 的数据还有一些没有被 follower 副本的同步的话，就会造成消息丢失。**

#### 设置 acks = all

解决办法就是我们设置  **acks = all**。acks 是 Kafka 生产者(Producer)  很重要的一个参数。

acks 的默认值即为1，代表我们的消息被leader副本接收之后就算被成功发送。当我们配置 **acks = all** 则代表所有副本都要接收到该消息之后该消息才算真正成功被发送。

#### 设置 replication.factor >= 3

为了保证 leader 副本能有 follower 副本能同步消息，我们一般会为 topic 设置 **replication.factor >= 3**。这样就可以保证每个 分区(partition) 至少有 3 个副本。虽然造成了数据冗余，但是带来了数据的安全性。

#### 设置 min.insync.replicas > 1

一般情况下我们还需要设置 **min.insync.replicas> 1** ，这样配置代表消息至少要被写入到 2 个副本才算是被成功发送。**min.insync.replicas** 的默认值为 1 ，在实际生产中应尽量避免默认值 1。

但是，为了保证整个 Kafka 服务的高可用性，你需要确保 **replication.factor > min.insync.replicas** 。为什么呢？设想一下加入两者相等的话，**只要是有一个副本挂掉，整个分区就无法正常工作了**。这明显违反高可用性！一般推荐设置成 **replication.factor = min.insync.replicas + 1**。

#### 设置 unclean.leader.election.enable = false

> **Kafka 0.11.0.0版本开始 unclean.leader.election.enable 参数的默认值由原来的true 改为false**

我们最开始也说了我们发送的消息会被发送到 leader 副本，然后 follower 副本才能从 leader 副本中拉取消息进行同步。多个 follower 副本之间的消息同步情况不一样，当我们配置了 **unclean.leader.election.enable = false**  的话，当 leader 副本发生故障时就不会从  follower 副本中和 leader 同步程度达不到要求的副本中选择出  leader ，这样降低了消息丢失的可能性。

### 2. Kafka处理重复消费

