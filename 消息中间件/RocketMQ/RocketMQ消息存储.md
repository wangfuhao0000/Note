## 存储概要设计

RocketMQ主要存储的文件包括Commitlog文件、ConsumeQueue文件、IndexFile文件。**RocketMQ将所有主题的消息存储到同一文件Commitlog中，确保消息发送时顺序写文件**，确保消息发送的高性能与高吞吐量。但是因为消息基本是按照主题进行消费，那我们要找指定主题的消息时，就需要遍历Commitlog，性能降低；为了提高消费效率，RocketMQ引入了ConsumeQueue消息队列文件，**每个消息主题包含多个消息消费队列，每个消息队列有一个消息文件**。IndexFile索引文件主要为了加速消息的检索性能，根据消息的属性快速从Commitlog文件中检索消息。结构图如下：

![rocketmq_design_1](/Users/quanzeng/SpringProjects/rocketmq/docs/cn/image/rocketmq_design_1.png)

1. Commitlog：消息存储文件，所有主题的消息都存储在这里。
2. ConsumeQueue：消息消费队列，消息到达Commitlog后，**将异步转发到消息消费队列**，供消息消费者消费。
3. IndexFile：消息索引文件，主要存储消息Key与Offset的对应关系。
4. 事务状态服务：存储每条消息的事务状态
5. 定时消息服务：每个延迟级别对应一个消息消费队列，存储延迟队列的消息拉取进度。（关于延迟级别后面再提）

## 初识消息存储

消息存储实现类：**org.apache.rocketmq.store.DefaultMessageStore**，包含了很多属性对存储文件操作的API，所以对文件操作实际就是对于它里面的属性进行操作，刚才提到的三个文件也都是作为它的属性存在的，看一下它的类图和核心属性：



- MessageStoreConfig：消息存储配置属性
- **CommitLog**：CommitLog文件的存储实现类
- **ConcurrentMap<String/* topic */, ConcurrentMap<Integer/* queueId */, ConsumeQueue>> consumeQueueTable**：消息队列存储缓存表，就是每个topic对应一个map，而map里面可以存储多个消息队列，所以就形成了这种一个topic对应多个消息队列的结构。
- FlushConsumeQueueService：消息队列文件ConsumeQueue刷盘线程
- CleanCommitLogService：清除Commitlog文件服务
- CleanConsumeQueueService：清除ConsumeQueue文件服务
- **IndexService**：索引文件实现类
- AllocateMappedService：Commitlog消息分发，根据Commitlog文件创建ConsumeQueue、IndexFile文件。
- HAService：存储HA机制，进行主从复制，后面文章会提到。
- TransientStorePool：消息堆内存缓存
- MessageArrivingListener：消息拉取长轮询模式消息到达监听器
- BrokerConfig：Broker配置属性
- StoreCheckpoint：文件刷盘检测点
- LinkedList<CommitLogDispatcher>：Commitlog文件转发请求

## 消息发送存储流程

首先看一下它的入口，就是DefaultMessageStore#putMessage方法。首先会检查当前Broker的状态和消息的格式是否符合要求，然后开始写入消息。如果消息写入失败了，则需要记录失败的次数进行自增。

```java
public PutMessageResult putMessage(MessageExtBrokerInner msg) {
  	//PutMessageStatus是个Enum类型，此方法会判断Broker是否停止工作；或Broker为SLAVE角色；
  	//或消息的长度进行判断：主题长度小于256字符，消息属性长度小于65536字符
  	//注意Broker的Slave角色是不支持写入消息的，只有Master才支持
    PutMessageStatus checkStoreStatus = this.checkStoreStatus();  
    if (checkStoreStatus != PutMessageStatus.PUT_OK) {   //不能写入，则返回结果状态
        return new PutMessageResult(checkStoreStatus, null);
    }

    PutMessageStatus msgCheckStatus = this.checkMessage(msg);  //消息长度检查
    if (msgCheckStatus == PutMessageStatus.MESSAGE_ILLEGAL) {
        return new PutMessageResult(msgCheckStatus, null);
    }
  
		//检查都通过了，开始准备写入，记录消息写之前和写完成后的时间
    long beginTime = this.getSystemClock().now();
  
  	//消息写入的真正实现，就是对于属性CommitLog的操作了，因为是存放到commitlog中
    PutMessageResult result = this.commitLog.putMessage(msg);  
  
    long elapsedTime = this.getSystemClock().now() - beginTime;
    if (elapsedTime > 500) {
        log.warn("not in lock elapsed time(ms)={}, bodyLength={}", elapsedTime, msg.getBody().length);
    }

    this.storeStatsService.setPutMessageEntireTimeMax(elapsedTime);

  	//如果result失败了，则记录存放消息失败的次数，让其加1
    if (null == result || !result.isOk()) {
        this.storeStatsService.getPutMessageFailedTimes().incrementAndGet();
    }

    return result;
}
```

接下来看看CommitLog是如何存储消息的，它的方法是CommitLog#putMessage。因为涉及到对于Message的各种设置，Message的属性可以看附录

```java
public PutMessageResult putMessage(final MessageExtBrokerInner msg) {
    // Set the storage time
    msg.setStoreTimestamp(System.currentTimeMillis());
    // Set the message body BODY CRC (consider the most appropriate setting
    // on the client)
    msg.setBodyCRC(UtilAll.crc32(msg.getBody()));
    // Back to Results
    AppendMessageResult result = null;

  	//注意在DefaultMessageStore初始化CommitLog时，就会将自己传进来，保证CommitLog也可以访问到
  	//DefaultMessageStore，并调用它的一些Service
    StoreStatsService storeStatsService = this.defaultMessageStore.getStoreStatsService();

    String topic = msg.getTopic();
    int queueId = msg.getQueueId();

    final int tranType = MessageSysFlag.getTransactionValue(msg.getSysFlag());
    if (tranType == MessageSysFlag.TRANSACTION_NOT_TYPE
        || tranType == MessageSysFlag.TRANSACTION_COMMIT_TYPE) {
        // Delay Delivery，看看消息的延迟级别，
      	//如果消息延迟级别大于0，则将消息的原主题和原ID放到消息的Property中
      	//然后用延迟消息主题SCHEDULE_TOPIC、消息队列ID更新原先消息的主题和队列
        if (msg.getDelayTimeLevel() > 0) {
            if (msg.getDelayTimeLevel() > this.defaultMessageStore.getScheduleMessageService().getMaxDelayLevel()) {
                msg.setDelayTimeLevel(this.defaultMessageStore.getScheduleMessageService().getMaxDelayLevel());
            }

            topic = ScheduleMessageService.SCHEDULE_TOPIC;
            queueId = ScheduleMessageService.delayLevel2QueueId(msg.getDelayTimeLevel());

            // 将自己真正的主题和ID放到属性中
            MessageAccessor.putProperty(msg, MessageConst.PROPERTY_REAL_TOPIC, msg.getTopic());
            MessageAccessor.putProperty(msg, MessageConst.PROPERTY_REAL_QUEUE_ID, String.valueOf(msg.getQueueId()));
            msg.setPropertiesString(MessageDecoder.messageProperties2String(msg.getProperties()));
						//设置主题和队列为延迟消息主题和延迟级别
            msg.setTopic(topic);
            msg.setQueueId(queueId);
        }
    }

  	//对于IPV6的支持
    InetSocketAddress bornSocketAddress = (InetSocketAddress) msg.getBornHost();
    if (bornSocketAddress.getAddress() instanceof Inet6Address) {
        msg.setBornHostV6Flag();
    }

    InetSocketAddress storeSocketAddress = (InetSocketAddress) msg.getStoreHost();
    if (storeSocketAddress.getAddress() instanceof Inet6Address) {
        msg.setStoreHostAddressV6Flag();
    }

  	//最后写入完成会记录占用写锁的时间，如果超过了500，则会进行相应的记录
    long elapsedTimeInLock = 0;

  	//得到最新的一个MappedFile，因为旧的肯定是已经写满了的，所以得到的是最后一个
  	//但最后一个也可能会因为剩余容量不足导致无法写入当前消息，所以就需要一个unlockMappedFile存储
  	//刚才的最后一个，然后去新建一个.
    MappedFile unlockMappedFile = null;
    MappedFile mappedFile = this.mappedFileQueue.getLastMappedFile();  

  	//保证消息的写入是串行的，先加锁。这个锁在初始化的时候会根据配置来决定是使用spin 还是 ReentrantLock。
    putMessageLock.lock(); 
    try {
        long beginLockTimestamp = this.defaultMessageStore.getSystemClock().now();
        this.beginTimeInLock = beginLockTimestamp;
        // Here settings are stored timestamp, in order to ensure an orderly
        // global
        msg.setStoreTimestamp(beginLockTimestamp);

        if (null == mappedFile || mappedFile.isFull()) {
          	// Mark: NewFile may be cause noise。可能是刚开始没有文件，所以将offset作为0创建新文件
            mappedFile = this.mappedFileQueue.getLastMappedFile(0); 
        }
        if (null == mappedFile) {   //创建失败了，返回错误信息
            log.error("create mapped file1 error, topic: " + msg.getTopic() + " clientAddr: " + msg.getBornHostString());
            beginTimeInLock = 0;
            return new PutMessageResult(PutMessageStatus.CREATE_MAPEDFILE_FAILED, null);
        }
				
      	//MappedFile添加消息的真正实现
        result = mappedFile.appendMessage(msg, this.appendMessageCallback);
        switch (result.getStatus()) {
            case PUT_OK:
                break;
            case END_OF_FILE:   //最后一个MappedFile的空间不足，需要新建一个
                unlockMappedFile = mappedFile;
                // Create a new file, re-write the message
                mappedFile = this.mappedFileQueue.getLastMappedFile(0);
                if (null == mappedFile) {
                    // XXX: warn and notify me
                    log.error("create mapped file2 error, topic: " + msg.getTopic() + " clientAddr: " + msg.getBornHostString());
                    beginTimeInLock = 0;
                    return new PutMessageResult(PutMessageStatus.CREATE_MAPEDFILE_FAILED, result);
                }
            		//针对新建的文件来写入消息
                result = mappedFile.appendMessage(msg, this.appendMessageCallback);
                break;
            case MESSAGE_SIZE_EXCEEDED:
            case PROPERTIES_SIZE_EXCEEDED:
                beginTimeInLock = 0;
                return new PutMessageResult(PutMessageStatus.MESSAGE_ILLEGAL, result);
            case UNKNOWN_ERROR:
                beginTimeInLock = 0;
                return new PutMessageResult(PutMessageStatus.UNKNOWN_ERROR, result);
            default:
                beginTimeInLock = 0;
                return new PutMessageResult(PutMessageStatus.UNKNOWN_ERROR, result);
        }

        elapsedTimeInLock = this.defaultMessageStore.getSystemClock().now() - beginLockTimestamp;
        beginTimeInLock = 0;
    } finally {
        putMessageLock.unlock();
    }

  	//根据持有锁的时间（其实就是将消息写入到文件中花费的时间）记录下来，做一个警告日志
    if (elapsedTimeInLock > 500) {
        log.warn("[NOTIFYME]putMessage in lock cost time(ms)={}, bodyLength={} AppendMessageResult={}", elapsedTimeInLock, msg.getBody().length, result);
    }

  	//处理刚才的unlockMappedFile
    if (null != unlockMappedFile && this.defaultMessageStore.getMessageStoreConfig().isWarmMapedFileEnable()) {
        this.defaultMessageStore.unlockMappedFile(unlockMappedFile);
    }

    PutMessageResult putMessageResult = new PutMessageResult(PutMessageStatus.PUT_OK, result);

    // Statistics
    storeStatsService.getSinglePutMessageTopicTimesTotal(msg.getTopic()).incrementAndGet();
    storeStatsService.getSinglePutMessageTopicSizeTotal(topic).addAndGet(result.getWroteBytes());
    //处理刷盘和主从复制
    handleDiskFlush(result, putMessageResult, msg);
    handleHA(result, putMessageResult, msg);

    return putMessageResult;
}
```

然后看看MappedFile#appendMessage的具体实现。要知道MappedFile其实就是一个具体的CommitLog文件，MappedFileQueue则是存储所有的MappedFile的地方，角色类似于一个目录。

```java
public AppendMessageResult appendMessagesInner(final MessageExt messageExt, final AppendMessageCallback cb) {
    assert messageExt != null;
    assert cb != null;

  	//当前MappedFile(commitlog)文件的写指针
    int currentPos = this.wrotePosition.get();  

    if (currentPos < this.fileSize) {
      	//如果writeBuffer为null，则证明数据都是直接放到mappedByteBuffer中
      	//所以调用slice()方法让byteBuffer和mappedByteBuffer共享缓冲区。
      	//否则就和writeBuffer共享缓冲区
        ByteBuffer byteBuffer = writeBuffer != null ? writeBuffer.slice() : this.mappedByteBuffer.slice();
        byteBuffer.position(currentPos);
        AppendMessageResult result;
      	//调用传进来的回调函数的doAppend方法，分为单个消息和批量消息的情况
        if (messageExt instanceof MessageExtBrokerInner) {  //单个msg文件
            result = cb.doAppend(this.getFileFromOffset(), byteBuffer, this.fileSize - currentPos, (MessageExtBrokerInner) messageExt);
        } else if (messageExt instanceof MessageExtBatch) {   //批量的msg文件
            result = cb.doAppend(this.getFileFromOffset(), byteBuffer, this.fileSize - currentPos, (MessageExtBatch) messageExt);
        } else {
            return new AppendMessageResult(AppendMessageStatus.UNKNOWN_ERROR);
        }
      	//本文件新加消息了，所以需要更新写指针，添加上写入消息的长度
      	//更新最后一次更新（存新消息）的时间戳
        this.wrotePosition.addAndGet(result.getWroteBytes());
        this.storeTimestamp = result.getStoreTimestamp();
        return result;
    }
    log.error("MappedFile.appendMessage return null, wrotePosition: {} fileSize: {}", currentPos, this.fileSize);
    return new AppendMessageResult(AppendMessageStatus.UNKNOWN_ERROR);
}
```

然后看一下回调函数中，如何进行文件的写入的。

```java
public AppendMessageResult doAppend(final long fileFromOffset, final ByteBuffer byteBuffer, final int maxBlank,
    final MessageExtBrokerInner msgInner) {
    // STORETIMESTAMP + STOREHOSTADDRESS + OFFSET <br>

    // PHY OFFSET
    long wroteOffset = fileFromOffset + byteBuffer.position();

    int sysflag = msgInner.getSysFlag();
    int bornHostLength = (sysflag & MessageSysFlag.BORNHOST_V6_FLAG) == 0 ? 4 + 4 : 16 + 4;
    int storeHostLength = (sysflag & MessageSysFlag.STOREHOSTADDRESS_V6_FLAG) == 0 ? 4 + 4 : 16 + 4;
    ByteBuffer bornHostHolder = ByteBuffer.allocate(bornHostLength);
    ByteBuffer storeHostHolder = ByteBuffer.allocate(storeHostLength);

    this.resetByteBuffer(storeHostHolder, storeHostLength);
    String msgId;
  	//创建全局唯一的msgId
    if ((sysflag & MessageSysFlag.STOREHOSTADDRESS_V6_FLAG) == 0) {
        msgId = MessageDecoder.createMessageId(this.msgIdMemory, msgInner.getStoreHostBytes(storeHostHolder), wroteOffset);
    } else {
        msgId = MessageDecoder.createMessageId(this.msgIdV6Memory, msgInner.getStoreHostBytes(storeHostHolder), wroteOffset);
    }

    // Record ConsumeQueue information
    keyBuilder.setLength(0);
    keyBuilder.append(msgInner.getTopic());
    keyBuilder.append('-');
    keyBuilder.append(msgInner.getQueueId());
    String key = keyBuilder.toString();  //消息的key为topic-queueId
    Long queueOffset = CommitLog.this.topicQueueTable.get(key);
    if (null == queueOffset) {
        queueOffset = 0L;
        CommitLog.this.topicQueueTable.put(key, queueOffset);
    }

    // Transaction messages that require special handling
    final int tranType = MessageSysFlag.getTransactionValue(msgInner.getSysFlag());
    switch (tranType) {
        // Prepared and Rollback message is not consumed, will not enter the
        // consumer queuec
        case MessageSysFlag.TRANSACTION_PREPARED_TYPE:
        case MessageSysFlag.TRANSACTION_ROLLBACK_TYPE:
            queueOffset = 0L;
            break;
        case MessageSysFlag.TRANSACTION_NOT_TYPE:
        case MessageSysFlag.TRANSACTION_COMMIT_TYPE:
        default:
            break;
    }

    /**
     * Serialize message
     */
    final byte[] propertiesData =
        msgInner.getPropertiesString() == null ? null : msgInner.getPropertiesString().getBytes(MessageDecoder.CHARSET_UTF8);

    final int propertiesLength = propertiesData == null ? 0 : propertiesData.length;

    if (propertiesLength > Short.MAX_VALUE) {
        log.warn("putMessage message properties length too long. length={}", propertiesData.length);
        return new AppendMessageResult(AppendMessageStatus.PROPERTIES_SIZE_EXCEEDED);
    }

    final byte[] topicData = msgInner.getTopic().getBytes(MessageDecoder.CHARSET_UTF8);
    final int topicLength = topicData.length;

    final int bodyLength = msgInner.getBody() == null ? 0 : msgInner.getBody().length;

    final int msgLen = calMsgLength(msgInner.getSysFlag(), bodyLength, topicLength, propertiesLength);

    // Exceeds the maximum message
    if (msgLen > this.maxMessageSize) {
        CommitLog.log.warn("message size exceeded, msg total size: " + msgLen + ", msg body size: " + bodyLength
            + ", maxMessageSize: " + this.maxMessageSize);
        return new AppendMessageResult(AppendMessageStatus.MESSAGE_SIZE_EXCEEDED);
    }

    // Determines whether there is sufficient free space
    if ((msgLen + END_FILE_MIN_BLANK_LENGTH) > maxBlank) {
        this.resetByteBuffer(this.msgStoreItemMemory, maxBlank);
        // 1 TOTALSIZE
        this.msgStoreItemMemory.putInt(maxBlank);
        // 2 MAGICCODE
        this.msgStoreItemMemory.putInt(CommitLog.BLANK_MAGIC_CODE);
        // 3 The remaining space may be any value
        // Here the length of the specially set maxBlank
        final long beginTimeMills = CommitLog.this.defaultMessageStore.now();
        byteBuffer.put(this.msgStoreItemMemory.array(), 0, maxBlank);
        return new AppendMessageResult(AppendMessageStatus.END_OF_FILE, wroteOffset, maxBlank, msgId, msgInner.getStoreTimestamp(),
            queueOffset, CommitLog.this.defaultMessageStore.now() - beginTimeMills);
    }

    // Initialization of storage space
    this.resetByteBuffer(msgStoreItemMemory, msgLen);
    // 1 TOTALSIZE
    this.msgStoreItemMemory.putInt(msgLen);
    // 2 MAGICCODE
    this.msgStoreItemMemory.putInt(CommitLog.MESSAGE_MAGIC_CODE);
    // 3 BODYCRC
    this.msgStoreItemMemory.putInt(msgInner.getBodyCRC());
    // 4 QUEUEID
    this.msgStoreItemMemory.putInt(msgInner.getQueueId());
    // 5 FLAG
    this.msgStoreItemMemory.putInt(msgInner.getFlag());
    // 6 QUEUEOFFSET
    this.msgStoreItemMemory.putLong(queueOffset);
    // 7 PHYSICALOFFSET
    this.msgStoreItemMemory.putLong(fileFromOffset + byteBuffer.position());
    // 8 SYSFLAG
    this.msgStoreItemMemory.putInt(msgInner.getSysFlag());
    // 9 BORNTIMESTAMP
    this.msgStoreItemMemory.putLong(msgInner.getBornTimestamp());
    // 10 BORNHOST
    this.resetByteBuffer(bornHostHolder, bornHostLength);
    this.msgStoreItemMemory.put(msgInner.getBornHostBytes(bornHostHolder));
    // 11 STORETIMESTAMP
    this.msgStoreItemMemory.putLong(msgInner.getStoreTimestamp());
    // 12 STOREHOSTADDRESS
    this.resetByteBuffer(storeHostHolder, storeHostLength);
    this.msgStoreItemMemory.put(msgInner.getStoreHostBytes(storeHostHolder));
    // 13 RECONSUMETIMES
    this.msgStoreItemMemory.putInt(msgInner.getReconsumeTimes());
    // 14 Prepared Transaction Offset
    this.msgStoreItemMemory.putLong(msgInner.getPreparedTransactionOffset());
    // 15 BODY
    this.msgStoreItemMemory.putInt(bodyLength);
    if (bodyLength > 0)
        this.msgStoreItemMemory.put(msgInner.getBody());
    // 16 TOPIC
    this.msgStoreItemMemory.put((byte) topicLength);
    this.msgStoreItemMemory.put(topicData);
    // 17 PROPERTIES
    this.msgStoreItemMemory.putShort((short) propertiesLength);
    if (propertiesLength > 0)
        this.msgStoreItemMemory.put(propertiesData);

    final long beginTimeMills = CommitLog.this.defaultMessageStore.now();
    // Write messages to the queue buffer
    byteBuffer.put(this.msgStoreItemMemory.array(), 0, msgLen);

    AppendMessageResult result = new AppendMessageResult(AppendMessageStatus.PUT_OK, wroteOffset, msgLen, msgId,
        msgInner.getStoreTimestamp(), queueOffset, CommitLog.this.defaultMessageStore.now() - beginTimeMills);

    switch (tranType) {
        case MessageSysFlag.TRANSACTION_PREPARED_TYPE:
        case MessageSysFlag.TRANSACTION_ROLLBACK_TYPE:
            break;
        case MessageSysFlag.TRANSACTION_NOT_TYPE:
        case MessageSysFlag.TRANSACTION_COMMIT_TYPE:
            // The next update ConsumeQueue information
            CommitLog.this.topicQueueTable.put(key, ++queueOffset);
            break;
        default:
            break;
    }
    return result;
}
```