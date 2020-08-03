## NameServer架构设计

Broker消息服务器启动时向所有NameServer注册，消息生产者发送消息前先从NameServer获取Broker服务器地址列表，然后根据负载算法从列表中选择一台消息服务器进行消息发送。NameServer与每台Broker保持长连接，并间隔30S检查Broker是否存活，检测到Broker宕机则将其从注册表中删除。**要注意NameServer之间互不通信**，会导致在某一时刻的数据并不会完全相同，但这不会对消息发送造成影响。

![rocketmq_architecture_1](/Users/quanzeng/SpringProjects/rocketmq/docs/cn/image/rocketmq_architecture_1.png)

## NameServer启动流程

NameServer启动类为：org.apache.rocketmq.namesrv.NamesrvStartup。大致流程如下：首先创建NamesrvController，它是NameServer的核心控制器类，然后进行初始化。

```java
public static void main(String[] args) {
    main0(args);
}

public static NamesrvController main0(String[] args) {

    try {
      	//1. 创建NameSrvController实例
        NamesrvController controller = createNamesrvController(args);  
      	//2. 进行初始化
        start(controller);
        String tip = "The Name Server boot success. serializeType=" + RemotingCommand.getSerializeTypeConfigInThisServer();
        log.info(tip);
        System.out.printf("%s%n", tip);
        return controller;
    } catch (Throwable e) {
        e.printStackTrace();
        System.exit(-1);
    }

    return null;
}
```

### 解析配置文件并创建NamesrvController实例

首先填充NameServerConfig、NettyServerConfig属性值。

```java
public static NamesrvController createNamesrvController(String[] args) throws IOException, JoranException {

    final NamesrvConfig namesrvConfig = new NamesrvConfig();
    final NettyServerConfig nettyServerConfig = new NettyServerConfig();
    nettyServerConfig.setListenPort(9876);  //设置监听端口默认为9876
  	//如果有配置文件，则将配置文件内的内容填充到两个config中
    if (commandLine.hasOption('c')) {
        String file = commandLine.getOptionValue('c');
        if (file != null) {
            InputStream in = new BufferedInputStream(new FileInputStream(file));
            properties = new Properties();
            properties.load(in);
            MixAll.properties2Object(properties, namesrvConfig);
            MixAll.properties2Object(properties, nettyServerConfig);

            namesrvConfig.setConfigStorePath(file);

            System.out.printf("load config properties file OK, %s%n", file);
            in.close();
        }
    }
  	//如果有 -p 选项，则表明只是打印配置内容，然后退出
    if (commandLine.hasOption('p')) {
        InternalLogger console = InternalLoggerFactory.getLogger(LoggerName.NAMESRV_CONSOLE_NAME);
        MixAll.printObjectProperties(console, namesrvConfig);
        MixAll.printObjectProperties(console, nettyServerConfig);
        System.exit(0);
    }
		//如果命令行指定了配置选项，则将内容填充到两个config中
    MixAll.properties2Object(ServerUtil.commandLine2Properties(commandLine), namesrvConfig);

  	//读取完两个config后，创建NameSrvController
    final NamesrvController controller = new NamesrvController(namesrvConfig, nettyServerConfig);

    // remember all configs to prevent discard
    controller.getConfiguration().registerConfig(properties);

    return controller;
}
```

### 初始化NamesrvController

创建完NameSrvController实例调用start(NamesrvController)并进行初始化。

```java
public static NamesrvController start(final NamesrvController controller) throws Exception {

    boolean initResult = controller.initialize();  //1. 调用controller的initialize方法进行初始化
    if (!initResult) {
        controller.shutdown();
        System.exit(-3);
    }
	
  	//注意这里有一个钩子函数，能让我们在JVM退出之前执行一些关闭自定义资源的操作。
    Runtime.getRuntime().addShutdownHook(new ShutdownHookThread(log, new Callable<Void>() {
        @Override
        public Void call() throws Exception {
            controller.shutdown();
            return null;
        }
    }));
    controller.start();   //2. 开启Controller

    return controller;
}
```

#### 初始化

初始化就是根据配置进行remoteServer的初始化，然后初始化自己的处理事件，然后有个方法registerProcessor，它是用来根据配置注册自己使用哪个线程池处理事件，最后开启两个定时任务：

- NameServer每隔10S扫描一次Broker，移除处于不激活状态的Broker
- NameServer每隔10分钟打印一次KV配置

```java
public boolean initialize() {
    this.kvConfigManager.load();
    this.remotingServer = new NettyRemotingServer(this.nettyServerConfig, this.brokerHousekeepingService);   //创建server
    this.remotingExecutor =
        Executors.newFixedThreadPool(nettyServerConfig.getServerWorkerThreads(), new ThreadFactoryImpl("RemotingExecutorThread_"));
    this.registerProcessor();  //注册server处理事件的线程

  	//每10s扫描一次Broker
    this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
        @Override
        public void run() {
            NamesrvController.this.routeInfoManager.scanNotActiveBroker();
        }
    }, 5, 10, TimeUnit.SECONDS);

  	//每10分钟打印一次配置
    this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
        @Override
        public void run() {
            NamesrvController.this.kvConfigManager.printAllPeriodically();
        }
    }, 1, 10, TimeUnit.MINUTES);

    return true;
}
```

#### 开启server

```java
public void start() throws Exception {
    this.remotingServer.start();   //就是开启remotingServer

    if (this.fileWatchService != null) {
        this.fileWatchService.start();
    }
}
```

## NameServer路由注册、故障剔除

NameServer的主要作用就是为消息生产者和消费者提供关于主题Topic的路由信息，所以它需要存储路由的基础信息，而且还可以管理Broker节点，例如路由注册和路由删除。

### 路由元信息

核心实现是RouteInfoManager类，看一下它存储哪些信息，每个结构的具体信息可以查看附录部分：

```java
//一个topic的消息可能存在于多个Broker中，使用此结构进行负载均衡，选择一个Broker进行消息的存储
private final HashMap<String/* topic */, List<QueueData>> topicQueueTable;
//一个brokerName代表一个Master-Slave架构，BrokerData定义看附录
private final HashMap<String/* brokerName */, BrokerData> brokerAddrTable;
//一个集群对应一堆Master-Slave架构
private final HashMap<String/* clusterName */, Set<String/* brokerName */>> clusterAddrTable;
//一个Broker实例对应一个LiveInfo，里面存储了一些配置消息。这里存储了所有的Broker实例信息
private final HashMap<String/* brokerAddr */, BrokerLiveInfo> brokerLiveTable;
//过滤相关信息，现在先不考虑
private final HashMap<String/* brokerAddr */, List<String>/* Filter Server */> filterServerTable;
```

这里要说一下，RocketMQ是基于订阅发布机制，**一个Topic有多个消息队列，一个Broker默认为每个Topic创建4个读队列和4个写队列**。多个Broker组成一个集群，多台BrokerName相同的Broker会组成一个Master-Slave架构，brokerId为0代表Master，大于0表示Slave。所以一个集群会由多个BrokerName互不相同的Master-Slave架构组成。

### 路由注册

RocketMQ路由注册是通过Broker与NameServer的心跳功能实现的。Broker启动时向集群中所有的NameServer发送心跳语句，每隔30S向集群中所有NameServer发送心跳包，NameServer收到Broker心跳包后会更新自己的路由元信息，其中包括**属性brokerLiveTable里的lastUpdateTimestamp**。然后NameServer每隔10S扫描BrokerLiveTable，如果连续120S没有收到心跳包，则NameServer会移除该Broker的路由信息，同时关闭对应的Socket连接。

#### Broker发送心跳包

```java
this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
    @Override
    public void run() {
        try {
            BrokerController.this.registerBrokerAll(true, false, brokerConfig.isForceRegister());
        } catch (Throwable e) {
            log.error("registerBrokerAll Exception", e);
        }
    }
}, 1000 * 10, Math.max(10000, Math.min(brokerConfig.getRegisterNameServerPeriod(), 60000)), TimeUnit.MILLISECONDS);
```

```java
    public List<RegisterBrokerResult> registerBrokerAll(
        final String clusterName,
        final String brokerAddr,
        final String brokerName,
        final long brokerId,
        final String haServerAddr,
        final TopicConfigSerializeWrapper topicConfigWrapper,
        final List<String> filterServerList,
        final boolean oneway,
        final int timeoutMills,
        final boolean compressed) {

        final List<RegisterBrokerResult> registerBrokerResultList = Lists.newArrayList();
      	//会向所有的NameServer进行注册
        List<String> nameServerAddressList = this.remotingClient.getNameServerAddressList();
        if (nameServerAddressList != null && nameServerAddressList.size() > 0) {
						//构建注册信息，也就是有关自己（Broker）的一些信息
          	//首先是头部
            final RegisterBrokerRequestHeader requestHeader = new RegisterBrokerRequestHeader();
            requestHeader.setBrokerAddr(brokerAddr);
            requestHeader.setBrokerId(brokerId);
            requestHeader.setBrokerName(brokerName);
            requestHeader.setClusterName(clusterName);
            requestHeader.setHaServerAddr(haServerAddr);
            requestHeader.setCompressed(compressed);
						//然后是注册消息的主体
            RegisterBrokerBody requestBody = new RegisterBrokerBody();
            requestBody.setTopicConfigSerializeWrapper(topicConfigWrapper);
            requestBody.setFilterServerList(filterServerList);
            final byte[] body = requestBody.encode(compressed);   //将消息体按照指定编码进行压缩
            final int bodyCrc32 = UtilAll.crc32(body);  //消息的CRC
            requestHeader.setBodyCrc32(bodyCrc32);
          
          	//保证所有的NameServer都发送并接收到响应后，才结束
            final CountDownLatch countDownLatch = new  CountDownLatch(nameServerAddressList.size());   
          
          	//给每个NameServer发送自己的注册信息
            for (final String namesrvAddr : nameServerAddressList) {
                brokerOuterExecutor.execute(new Runnable() {
                    @Override
                    public void run() {
                        try {
                          	//将消息发送到对应的NameServer上，底层使用的netty来传输
                            RegisterBrokerResult result = registerBroker(namesrvAddr,oneway, timeoutMills,requestHeader,body);
                            if (result != null) {  //将每个NameServer的响应结果都存储起来
                                registerBrokerResultList.add(result);
                            }
                            log.info("register broker[{}]to name server {} OK", brokerId, namesrvAddr);
                        } catch (Exception e) {
                            log.warn("registerBroker Exception, {}", namesrvAddr, e);
                        } finally {
                            countDownLatch.countDown();
                        }
                    }
                });
            }
						
          	//等到所有的都发送完成后，才返回到这个地方
            try {
                countDownLatch.await(timeoutMills, TimeUnit.MILLISECONDS);
            } catch (InterruptedException e) {
            }
        }
        return registerBrokerResultList;
    }
```

#### NameServer处理心跳包

org.apache.rocketmq.namesrv.processor.DefaultRequestProcessor网络处理器解析请求类型，如果请求类型是RequestCode.REGISTER_BROKER，则请求最终转发到RouteInfoManager#registerBroker。在注册前需要进行读锁的加锁，注册完成后需要释放读锁。主要包含以下几个步骤：

1. 首先判断Broker所属集群是否存在，如果不存在则创建，然后将Broker名加入到集群Broker集合中。
2. 然后看看此集群中有没有相同BrokerName的Master-Slave架构存在，没有则进行创建并加入到架构中。
3. 

```java
public RegisterBrokerResult registerBroker(
    final String clusterName,
    final String brokerAddr,
    final String brokerName,
    final long brokerId,
    final String haServerAddr,
    final TopicConfigSerializeWrapper topicConfigWrapper,
    final List<String> filterServerList,
    final Channel channel) {
    RegisterBrokerResult result = new RegisterBrokerResult();
    try {
        try {
            this.lock.writeLock().lockInterruptibly();  //先申请写锁
						//根据集群找到集群内所有的BrokerName，其实每个BrokerName对应的是个Master-Slave架构
            Set<String> brokerNames = this.clusterAddrTable.get(clusterName);
            if (null == brokerNames) {   //还没有这个集群，则创建一个集群，里面包含当前的brokerName
                brokerNames = new HashSet<String>();
                this.clusterAddrTable.put(clusterName, brokerNames);
            }
            brokerNames.add(brokerName);

            boolean registerFirst = false;
						//BrokerData代表的是Master—Slave架构的实际存储结构，查看有没有这个结构
            BrokerData brokerData = this.brokerAddrTable.get(brokerName);
            if (null == brokerData) {
                registerFirst = true;  //说明是第一次注册，然后注册一个Master-Slave架构并存储起来
                brokerData = new BrokerData(clusterName, brokerName, new HashMap<Long, String>());
                this.brokerAddrTable.put(brokerName, brokerData);
            }
          	//获得此Master-Slave中的所有Broker实例
            Map<Long, String> brokerAddrsMap = brokerData.getBrokerAddrs();
            //Switch slave to master: first remove <1, IP:PORT> in namesrv, then add <0, IP:PORT>
            //The same IP:PORT must only have one record in brokerAddrTable
            Iterator<Entry<Long, String>> it = brokerAddrsMap.entrySet().iterator();
            while (it.hasNext()) {
                Entry<Long, String> item = it.next();
              	//brokerAddr的key为BrokerID，value为Address，所以当Address相同且ID不同，
              	//则表示产生了冲突，有了重复的Broker，则删除旧的Broker
                if (null != brokerAddr && brokerAddr.equals(item.getValue()) && brokerId != item.getKey()) {
                    it.remove();
                }
            }
						//
            String oldAddr = brokerData.getBrokerAddrs().put(brokerId, brokerAddr);
            registerFirst = registerFirst || (null == oldAddr);

            if (null != topicConfigWrapper
                && MixAll.MASTER_ID == brokerId) {
                if (this.isBrokerTopicConfigChanged(brokerAddr, topicConfigWrapper.getDataVersion())
                    || registerFirst) {
                    ConcurrentMap<String, TopicConfig> tcTable =
                        topicConfigWrapper.getTopicConfigTable();
                    if (tcTable != null) {
                        for (Map.Entry<String, TopicConfig> entry : tcTable.entrySet()) {
                            this.createAndUpdateQueueData(brokerName, entry.getValue());
                        }
                    }
                }
            }

          	//更新BrokerLiveInfo信息，即更新每个Broker的最近收到心跳包的时间
            BrokerLiveInfo prevBrokerLiveInfo = this.brokerLiveTable.put(brokerAddr,
                new BrokerLiveInfo(
                    System.currentTimeMillis(),   //此Broker最近收到的心跳包的事件
                    topicConfigWrapper.getDataVersion(),
                    channel,
                    haServerAddr));
            if (null == prevBrokerLiveInfo) {   //说明是第一次注册
                log.info("new broker registered, {} HAServer: {}", brokerAddr, haServerAddr);
            }

          	//路由过滤相关的
            if (filterServerList != null) {   
                if (filterServerList.isEmpty()) {
                    this.filterServerTable.remove(brokerAddr);
                } else {
                    this.filterServerTable.put(brokerAddr, filterServerList);
                }
            }

            if (MixAll.MASTER_ID != brokerId) {
                String masterAddr = brokerData.getBrokerAddrs().get(MixAll.MASTER_ID);
                if (masterAddr != null) {
                    BrokerLiveInfo brokerLiveInfo = this.brokerLiveTable.get(masterAddr);
                    if (brokerLiveInfo != null) {
                        result.setHaServerAddr(brokerLiveInfo.getHaServerAddr());
                        result.setMasterAddr(masterAddr);
                    }
                }
            }
        } finally {
            this.lock.writeLock().unlock();
        }
    } catch (Exception e) {
        log.error("registerBroker Exception", e);
    }

    return result;
}
```

### 路由删除

NameServer会每隔10S扫描brokerLiveTable状态表，如果BrokerLive的lastUpdateTimestamp的时间戳（上次收到该Broker的心跳包）距离当前时间超过120S，则认为Broker失效，移除该Broker，关闭与Broker的连接，并同时更新topicQueueTable、brolerAddrTable、brokerLiveTable、filterServerTable。触发路由删除的操作有两种情况：

- 刚才说的扫描的情况，如果从上次收到某个Broker心跳包距离现在已经有120S，则进行剔除
- Broker正常关闭下，会执行unregisterBroker指令来删除



NameServer每10S执行一次scanNotActiveBroker函数，用来剔除失效的Broker实例。首先会将brokerLiveTable中的实例移除掉，因为这个表示表示一个broker实例是否存活的根本依据，且里面存储的对应的Channel。主要更新以下信息：

- 根据brokerAddress（代表了唯一的一个Broker实例）从brokerLiveTable中移除
- 维护brokerAddrTable。即更改此Broker实例所在的Master-Slave架构。如果移除后此架构没有Broker实例了，则也会移除此架构。
- 维护clusterAddrTable。根据所在的架构名字，找到它所在的集群然后进行信息更改，表明此集群中没有此架构了。如果移除此架构后该集群没有其他的架构了，则也会移除该集群。
- 维护主题队列topicQueueTable。因为每个主题可能会将消息存储在要删除的Broker上，如果删除此Broker后该Topic没有其余的Broker供使用了，则删除此Topic。

```java
public void scanNotActiveBroker() {
    Iterator<Entry<String, BrokerLiveInfo>> it = this.brokerLiveTable.entrySet().iterator();
    while (it.hasNext()) {
        Entry<String, BrokerLiveInfo> next = it.next();
        long last = next.getValue().getLastUpdateTimestamp();
        if ((last + BROKER_CHANNEL_EXPIRED_TIME) < System.currentTimeMillis()) {
          	//关闭Broker实例对应的Channel，并删除此实例
            RemotingUtil.closeChannel(next.getValue().getChannel());  
            it.remove();
            log.warn("The broker channel expired, {} {}ms", next.getKey(), BROKER_CHANNEL_EXPIRED_TIME);
          	//剩下的就是根据这个Broker实例来进行其他路由表的维护工作
            this.onChannelDestroy(next.getKey(), next.getValue().getChannel());
        }
    }
}


public void onChannelDestroy(String remoteAddr, Channel channel) {
        String brokerAddrFound = null;
  			//首先是根据channel在brokerLiveTable中找到对应的Broker实例地址。
  			//因为我们上面先执行的ite.remove()删除此实例，若还能根据channel从brokerLiveTable中找到
  			//说明我们没有删除成功
        if (channel != null) {
            try {
                try {
                    this.lock.readLock().lockInterruptibly();
                    Iterator<Entry<String, BrokerLiveInfo>> itBrokerLiveTable =
                        this.brokerLiveTable.entrySet().iterator();
                    while (itBrokerLiveTable.hasNext()) {
                        Entry<String, BrokerLiveInfo> entry = itBrokerLiveTable.next();
                        if (entry.getValue().getChannel() == channel) {
                            brokerAddrFound = entry.getKey();
                            break;
                        }
                    }
                } finally {
                    this.lock.readLock().unlock();
                }
            } catch (Exception e) {
                log.error("onChannelDestroy Exception", e);
            }
        }

  			//没找到，则说明我们已经清除成功了，然后只需要根据remoteAddr更新路由表就好了
        if (null == brokerAddrFound) {   
            brokerAddrFound = remoteAddr;
        } else {  //否则没有删除成功
            log.info("the broker's channel destroyed, {}, clean it's data structure at once", brokerAddrFound);
        }

        if (brokerAddrFound != null && brokerAddrFound.length() > 0) {
            try {
                try {
                    this.lock.writeLock().lockInterruptibly();
                  	//1. 从brokerLiveTable中移除
                    this.brokerLiveTable.remove(brokerAddrFound);  
                    this.filterServerTable.remove(brokerAddrFound);
                    String brokerNameFound = null;
                    boolean removeBrokerName = false;
                  	
                  	//2. 从brokerAddrTable中移除有关此Broker实例的信息
                    Iterator<Entry<String, BrokerData>> itBrokerAddrTable =
                        this.brokerAddrTable.entrySet().iterator();
                  	//遍历集群下所有的Master-Slave架构
                    while (itBrokerAddrTable.hasNext() && (null == brokerNameFound)) {
                      	//得到一个Master-Slave架构，遍历此结构看看本Broker实例是否存在于此架构中
                        BrokerData brokerData = itBrokerAddrTable.next().getValue();

                        Iterator<Entry<Long, String>> it = brokerData.getBrokerAddrs().entrySet().iterator();
                        while (it.hasNext()) {
                            Entry<Long, String> entry = it.next();
                            Long brokerId = entry.getKey();
                            String brokerAddr = entry.getValue();
                          	//找到了此Broker实例，进行删除并标记brokerNameFound
                            if (brokerAddr.equals(brokerAddrFound)) {   
                                brokerNameFound = brokerData.getBrokerName();
                                it.remove();
                                log.info("remove brokerAddr[{}, {}] from brokerAddrTable, because channel destroyed",
                                    brokerId, brokerAddr);
                                break;
                            }
                        }
												//此Master-Slave架构已经没有Broker实例了，则从集群中删除此架构
                        if (brokerData.getBrokerAddrs().isEmpty()) {
                            removeBrokerName = true;
                            itBrokerAddrTable.remove();
                            log.info("remove brokerName[{}] from brokerAddrTable, because channel destroyed",
                                brokerData.getBrokerName());
                        }
                    }
										
                  	//3. 第一个说明确实删除了一个Broker实例，第二个说明删除实例后导致此架构没有Broker实例了
                  	//其实就是刚才删除的Broker实例是此架构的最后一个Broker了，则需要从集群中删除此架构
                    if (brokerNameFound != null && removeBrokerName) {
                        Iterator<Entry<String, Set<String>>> it = this.clusterAddrTable.entrySet().iterator();
                        while (it.hasNext()) {  //遍历所有的集群
                            Entry<String, Set<String>> entry = it.next();
                            String clusterName = entry.getKey();
                            Set<String> brokerNames = entry.getValue();
                          	//尝试从某一个集群中删除此架构，这又会出现这个集群中只有这一个架构的问题
                          	//所以也会判断是不是最后一个架构，如果是则需要删除所在的集群
                            boolean removed = brokerNames.remove(brokerNameFound);
                            if (removed) {
                                log.info("remove brokerName[{}], clusterName[{}] from clusterAddrTable, because channel destroyed",
                                    brokerNameFound, clusterName);

                                if (brokerNames.isEmpty()) { 
                                    log.info("remove the clusterName[{}] from clusterAddrTable, because channel destroyed and no broker in this cluster",
                                        clusterName);
                                    it.remove();   //集群里面没有架构了，删除集群
                                }

                                break;
                            }
                        }
                    }

                  	//4. 更新topicQueueTable，因为某个topic就可能存在于刚刚删除的Broker实例中
                    if (removeBrokerName) {
                        Iterator<Entry<String, List<QueueData>>> itTopicQueueTable =
                            this.topicQueueTable.entrySet().iterator();
                        while (itTopicQueueTable.hasNext()) {
                            Entry<String, List<QueueData>> entry = itTopicQueueTable.next();
                            String topic = entry.getKey();
                            List<QueueData> queueDataList = entry.getValue();
                          	
                            Iterator<QueueData> itQueueData = queueDataList.iterator();
                            while (itQueueData.hasNext()) {
                                QueueData queueData = itQueueData.next();
                              	//如果这个topic对应的Broker是刚刚删除的Broker，则进行移除
                                if (queueData.getBrokerName().equals(brokerNameFound)) {
                                    itQueueData.remove();
                                    log.info("remove topic[{} {}], from topicQueueTable, because channel destroyed",
                                        topic, queueData);
                                }
                            }

                          	//如果这个 Topic没有了对应的Broker，则将这个Topic也进行移除
                            if (queueDataList.isEmpty()) {
                                itTopicQueueTable.remove();
                                log.info("remove topic[{}] all queue, from topicQueueTable, because channel destroyed",
                                    topic);
                            }
                        }
                    }
                } finally {
                    this.lock.writeLock().unlock();  //释放写锁
                }
            } catch (Exception e) {
                log.error("onChannelDestroy Exception", e);
            }
        }
    }
```

### 路由发现

RocketMQ的路由发现是非实时的，当Topic路由出现变化后，NameServer不主动推送给客户端，而是由客户端去定时主动获取最新的路由信息。客户端根据主题名称拉去路由信息，主题名为TOPIC_ROUTEINFO_BY_TOPIC。RocketMQ路有结果包含如下信息：

- orderTopicConf：顺序消息配置内容，来自于kvConfig
- List<QueueData> queueData：Topic队列元数据，也就是当前Topic在哪些Broker架构中，每个QueueData代表一个架构
- List<BrokerData> brokerDatas：topic分布的Broker元数据，每个BrokerData代表了具体的Broker架构信息。
- HashMap<String/* brokerAddress*/, List<String> /*filterServer*/>>：broker上过滤服务器地址列表

NameServer返回路由信息是通过类DefaultRequestProcessor#getRouteInfoByTopic实现的：

```java
public RemotingCommand getRouteInfoByTopic(ChannelHandlerContext ctx,
    RemotingCommand request) throws RemotingCommandException {
    final RemotingCommand response = RemotingCommand.createResponseCommand(null);
    final GetRouteInfoRequestHeader requestHeader =
        (GetRouteInfoRequestHeader) request.decodeCommandCustomHeader(GetRouteInfoRequestHeader.class);  

    TopicRouteData topicRouteData = this.namesrvController.getRouteInfoManager().pickupTopicRouteData(requestHeader.getTopic());

    if (topicRouteData != null) {
        if (this.namesrvController.getNamesrvConfig().isOrderMessageEnable()) {
            String orderTopicConf =
                this.namesrvController.getKvConfigManager().getKVConfig(NamesrvUtil.NAMESPACE_ORDER_TOPIC_CONFIG,
                    requestHeader.getTopic());
            topicRouteData.setOrderTopicConf(orderTopicConf);
        }

        byte[] content = topicRouteData.encode();
        response.setBody(content);
        response.setCode(ResponseCode.SUCCESS);
        response.setRemark(null);
        return response;
    }

    response.setCode(ResponseCode.TOPIC_NOT_EXIST);
    response.setRemark("No topic route info in name server for the topic: " + requestHeader.getTopic()
        + FAQUrl.suggestTodo(FAQUrl.APPLY_TOPIC_URL));
    return response;
}
```



## 附录

#### NameServerConfig属性

```java
    private String rocketmqHome = System.getProperty(MixAll.ROCKETMQ_HOME_PROPERTY, System.getenv(MixAll.ROCKETMQ_HOME_ENV));   //rocketmq主目录
    private String kvConfigPath = System.getProperty("user.home") + File.separator + "namesrv" + File.separator + "kvConfig.json";  //存储KV配置属性的持久化路径
    private String configStorePath = System.getProperty("user.home") + File.separator + "namesrv" + File.separator + "namesrv.properties";  //默认配置文件路径
    private String productEnvName = "center";
    private boolean clusterTest = false;
    private boolean orderMessageEnable = false;  //是否支持顺序消费，默认是不支持
```

#### NettyServerConfig属性

要知道Netty会根据业务类型创建不同的线程池，例如处理消息发送、消息消费等，但如果一个业务没有注册线程池，则会使用public线程池执行，也就是属性serverCallbackExecutorThreads对应的线程池。

```java
private int listenPort = 8888;  //监听端口，初始化默认是9876
private int serverWorkerThreads = 8;   //Netty业务线程池个数
private int serverCallbackExecutorThreads = 0;   //public线程池线程数
private int serverSelectorThreads = 3;
private int serverOnewaySemaphoreValue = 256;
private int serverAsyncSemaphoreValue = 64;
private int serverChannelMaxIdleTimeSeconds = 120;

private int serverSocketSndBufSize = NettySystemConfig.socketSndbufSize;
private int serverSocketRcvBufSize = NettySystemConfig.socketRcvbufSize;
private boolean serverPooledByteBufAllocatorEnable = true;
```

#### QueueData

每个Topic会对应一个List<QueueData>，它代表每个Topic的信息会存在于多个Broker架构中，而每个QueueData就代表了一个Broker架构的信息，其属性如下：

```java
private String brokerName;   //broker名字，或者说是Broker所在的Master-slave架构
private int readQueueNums;   //读队列数量
private int writeQueueNums;  //写队列数量
private int perm;   //读写权限
private int topicSynFlag;   //topic同步标记
```

#### BrokerData

每个BrokerName对应一个BrokerData，其实此结构就代表了一个Broker的Master-Slave架构，其中维护了一个Map代表包含的所有Broker实例，key为实例ID，value为实例IP地址，其中ID为0的代表角色是Master，否则为Salve。其属性如下：

```java
private String cluster;   //集群名字，因为一个集群可能有多个Master-Slave架构
private String brokerName;  
private HashMap<Long/* brokerId */, String/* broker address */> brokerAddrs; //存储此架构所有的Broker
```

#### BrokerLiveInfo

一个Broker实例（Broker的IP地址）对应一个此结构，它具体的存储了每个Broker的详细信息。

```java
private long lastUpdateTimestamp;  //上次收到此Broker心跳包的时间
private DataVersion dataVersion;   //此Broker的数据版本？
private Channel channel;           //此Broker对应的Channel，通过它可以进行数据交互
private String haServerAddr;       //用于主从复制的
```