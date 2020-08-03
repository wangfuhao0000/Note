### AbstractLoadBalancer

首先看看类`AbstractLoadBalancer`，它是`ILoadBalancer`接口的抽象实现，在这个类中定义了一个关于服务实例的分组枚举类ServerGroup，它包含以下三种不同类型：

- ALL：所有服务的实例
- STATUS_UP：正常服务的实例
- STATUS_NOT_UP：停止服务的实例

它选择服务器的方法是`chooseServer()`函数，最终调用的是接口中的`chooseServer(Object key)`实现，其中参数为`null`表示选择具体服务实例时忽略key的条件判断。

```java
public abstract class AbstractLoadBalancer implements ILoadBalancer {
    public AbstractLoadBalancer() {
    }

    public Server chooseServer() {
        return this.chooseServer((Object)null);
    }
  
		//根据分组类型来获取对应的服务器实例
    public abstract List<Server> getServerList(AbstractLoadBalancer.ServerGroup var1);

  	//获取LoadBalancerStats对象，此对象存储负载均衡器中各个服务实例当前的属性和统计信息
  	//这些信息对于制定负载均衡策略很有用
    public abstract LoadBalancerStats getLoadBalancerStats();

    public static enum ServerGroup {
        ALL,
        STATUS_UP,
        STATUS_NOT_UP;

        private ServerGroup() {
        }
    }
}
```

### BaseLoadBalancer

这个类是Ribbon负载均衡器的基础实现类，该类定义中的主要内容包括：

- 定义并维护两个存储服务实例Server对象的列表，一个用来存储**所有服务**实例的清单，一个用于存储**正常服务**的实例清单。

```java
@Monitor(name = "LoadBalancer_AllServerList", type = DataSourceType.INFORMATIONAL)
protected volatile List<Server> allServerList;
@Monitor(name = "LoadBalancer_UpServerList", type = DataSourceType.INFORMATIONAL)
protected volatile List<Server> upServerList;
```

- 定义了刚才说的用来存储各服务实例属性和统计信息的`LoadBalancerStats`对象
- 定义了检查服务实例是否正常服务的IPing对象，需要在构造时注入它的具体实现
- 定义了检查服务实例操作的执行策略`IPingStrategy`，在`BaseLoadBalancer`中使用的是静态内部类`SerialPingStrategy`。它实现的策略就是线性的对每个Server进行Ping来检查服务器实例的状态。

```java
private static class SerialPingStrategy implements IPingStrategy {
    private SerialPingStrategy() {
    }

    public boolean[] pingServers(IPing ping, Server[] servers) {
        int numCandidates = servers.length;
        boolean[] results = new boolean[numCandidates];   //存储每个服务器的状态

        for(int i = 0; i < numCandidates; ++i) {  //线性的去Ping每个服务实例
            results[i] = false;
            try {
                if (ping != null) {
                    results[i] = ping.isAlive(servers[i]);  
                }
            } catch (Throwable var7) {
                BaseLoadBalancer.logger.error("Exception while pinging Server:" + servers[i], var7);
            }
        }

        return results;
    }
}
```

- 定义了负载均衡的处理规则`IRule`对象，负载均衡器是将服务实例选择任务委托给了`IRule`实例中的`choose()`函数实现。且在这里默认初始化`RoundRobinRule`为`IRule`的实现对象。`RoundRobinRule`实现了最基本且最常用的**线性负载均衡**规则，注意这里和上面说的不一样，上面那个是用来检测服务器实例状态的，而这里是用来选择一个具体的服务器实例来执行任务的。

```java
public Server choose(ILoadBalancer lb, Object key) {
    if (lb == null) {
        log.warn("no load balancer");
        return null;
    } else {
        Server server = null;
        int count = 0;

        while(true) {
            if (server == null && count++ < 10) {
                List<Server> reachableServers = lb.getReachableServers();  //得到所有的可达的Server
                List<Server> allServers = lb.getAllServers();
                int upCount = reachableServers.size();
                int serverCount = allServers.size();
                if (upCount != 0 && serverCount != 0) {
                  	//使用自定义的CAS来选择服务实例下标
                    int nextServerIndex = this.incrementAndGetModulo(serverCount);
                    server = (Server)allServers.get(nextServerIndex);
                    if (server == null) {
                        Thread.yield();
                    } else {
                        if (server.isAlive() && server.isReadyToServe()) {
                            return server;
                        }

                        server = null;
                    }
                    continue;
                }

                log.warn("No up servers available from load balancer: " + lb);
                return null;
            }

            if (count >= 10) {
                log.warn("No available alive servers after 10 tries from load balancer: " + lb);
            }

            return server;
        }
    }
}
```

- 启动ping任务，在`BaseLoadBalancer`的默认构造函数中，会直接启动一个定时检查Server是否健康的任务，间隔为10S。通过函数`setupPingTask()`方法来开启。

```java
void setupPingTask() {
    if (!this.canSkipPing()) {
        if (this.lbTimer != null) {
            this.lbTimer.cancel();
        }

        this.lbTimer = new ShutdownEnabledTimer("NFLoadBalancer-PingTimer-" + this.name, true);
      	//pingIntervalSeconds默认为10
        this.lbTimer.schedule(new BaseLoadBalancer.PingTask(), 0L, (long)(this.pingIntervalSeconds * 1000));
        this.forceQuickPing();
    }
}
```

- 当然实现了`ILoadBalancer`接口定义的负载均衡器需要实现其中定义的方法，包括：
  - addServers(List newServers)：这个方法是先获得原来的服务器实例，然后将newServers添加到里面，最后通过`setServersList`函数来覆盖旧的列表。
  - chooseServer(Object key)：挑选一个具体的服务实例
  - markServerDown(Server server)：标记某个服务实例暂停服务，通过调用`server.setAlive(false)`方法
  - getReachableServers()：获取可用的服务实例列表，因为此类已经维护了一个List，所以直接返回即可
  - getAllServers()：和上面一样，由于已经维护了，所以也可以直接返回。

### DynamicServerListLoadBalancer

此类继承自`BaseLoadBalancer`，它是对基础负载均衡器的扩展。此负载均衡器中实现类服务实例清单在运行期的**动态更新能力**；同时还具备对服务实例清单的**过滤功能**。即我们可通过过滤器来选择性的获取一批服务实例清单。接下来看看它添加了什么内容。

#### ServerList（动态更新能力）

首先我们能看到新增加了一个关于服务列表的操作对象`ServerList<T> serverListImpl`，此接口定义如下：

```java
public interface ServerList<T extends Server> {
    List<T> getInitialListOfServers();  //用于获取初始化的服务实例清单

    List<T> getUpdatedListOfServers();  //用于获取更新的服务实例清单
}
```

然后我们去查找实际是由哪个类来实现这个接口提供服务的，通过找到配置类`EurekaRibbonClientConfiguration`找到了具体实现。看到这里虽然返回的是个`DomainExtractingServerList`，但它实现的上述两个方法都是委托给了内部的一个`ServerList`，而内部的这个`ServerList`正是刚开始新建并传入到构造参数中的`DiscoveryEnabledNIWSServerList`，所以最终的实现或者说最后调用的还是`DiscoveryEnabledNIWSServerList`里面的两个方法。

```java
@Bean
@ConditionalOnMissingBean
public ServerList<?> ribbonServerList(IClientConfig config) {
  	//先创建一个DiscoveryEnabledNIWSServerList对象
  	//然后将其作为参数传入到DomainExtractingServerList的构造函数中
  	//最后返回的是DomainExtractingServerList。
    DiscoveryEnabledNIWSServerList discoveryServerList = new DiscoveryEnabledNIWSServerList(config);
    DomainExtractingServerList serverList = new DomainExtractingServerList(discoveryServerList, config, this.approximateZoneFromHostname);
    return serverList;
}
```

那么我们就具体看一下`DiscoveryEnabledNIWSServerList`是如何实现这两个方法来获取服务实例的，从源码能看出都是通过一个私有函数`obtainServersViaDiscovery()`来获取的。能看到它主要是**依靠EurekaClient从服务注册中心获取到具体的服务实例InstanceInfo列表**。

```java
private List<DiscoveryEnabledServer> obtainServersViaDiscovery() {
    List<DiscoveryEnabledServer> serverList = new ArrayList();
    if (this.eurekaClientProvider != null && this.eurekaClientProvider.get() != null) {
      	//先得到一个EurekaClient客户端
        EurekaClient eurekaClient = (EurekaClient)this.eurekaClientProvider.get();
        if (this.vipAddresses != null) {
            String[] arr$ = this.vipAddresses.split(",");
            int len$ = arr$.length;

            for(int i$ = 0; i$ < len$; ++i$) {
                String vipAddress = arr$[i$];
              	//调用client.getInstanceByVipAddress方法来获取一个服务对应的所有实例的信息
              	//这里的vipAddress就是逻辑的服务名字，比如USER-SERVICE
                List<InstanceInfo> listOfInstanceInfo = eurekaClient.getInstancesByVipAddress(vipAddress, this.isSecure, this.targetRegion);
                Iterator i$ = listOfInstanceInfo.iterator();
			
                while(i$.hasNext()) {
                    InstanceInfo ii = (InstanceInfo)i$.next();
                  	//对该服务对应的所有实例进行遍历
                  	//如果服务实例的状态是UP，则将其转换为DiscoveryEnabledServer对象
                  	//并返回所有服务实例的列表
                    if (ii.getStatus().equals(InstanceStatus.UP)) {
                        if (this.shouldUseOverridePort) {
                            if (logger.isDebugEnabled()) {
                                logger.debug("Overriding port on client name: " + this.clientName + " to " + this.overridePort);
                            }

                            InstanceInfo copy = new InstanceInfo(ii);
                            if (this.isSecure) {
                                ii = (new Builder(copy)).setSecurePort(this.overridePort).build();
                            } else {
                                ii = (new Builder(copy)).setPort(this.overridePort).build();
                            }
                        }

                      	//将状态为UP的服务实例进行封装，并记录起来，最后返回
                        DiscoveryEnabledServer des = new DiscoveryEnabledServer(ii, this.isSecure, this.shouldUseIpAddr);
                        des.setZone(DiscoveryClient.getZone(ii));
                        serverList.add(des);  
                    }
                }

                if (serverList.size() > 0 && this.prioritizeVipAddressBasedServers) {
                    break;
                }
            }
        }

        return serverList;
    } else {
        logger.warn("EurekaClient has not been initialized yet, returning an empty list");
        return new ArrayList();
    }
}
```

#### ServerListUpdater（触发更新并进行更新）

上面我们说的是从Eureka Server获取服务实例清单，那么这个动作触发是在哪，以及获得了服务实例清单后如何更新本地的服务实例清单呢。就是通过ServerListUpdater来实现的。



#### ServerListFilter

此接口非常简单，定义了方法`getFilteredListOfServers(List<T> var1)`用于实现对服务实例列表的过滤，也就是将传入的服务实例列表根据一些规则进行过滤，然后再返回。

```java
public interface ServerListFilter<T extends Server> {
    List<T> getFilteredListOfServers(List<T> var1);
}
```

针对于该接口的实现，除了`ZonePreferenceServerListFilter`的实现是Spring Cloud的扩展外，其他都是Netflix Ribbon中的原生实现类，我们直接看看这些过滤器的特点吧：

- AbstractServerListFilter：是个抽象过滤器，它定义了过滤时需要的一个重要依据对象`LoadBalancerStates`，我们在`AbstractLoadBalancer`中说过，它存储了关于负载均衡器的一些属性和统计信息等。

```java
public abstract class AbstractServerListFilter<T extends Server> implements ServerListFilter<T> {
    private volatile LoadBalancerStats stats;  //定义了一个LoadBalancerStats对象

    public AbstractServerListFilter() {
    }
  	//getter和setter
}
```

- ZoneAffinityServerListFilter：该过滤器基于“区域感知”的方式实现服务实例的过滤，其实就是根据服务实例所在的区域（Zone）和消费者自己所处的区域（Zone）进行比较，过滤那些不是同处一个区域的实例。

### ZoneAwareLoadBalancer

它是对`DynamicServerListLoadBalancer`的扩展，因为在`DynamicServerListLoadBalancer`没有重写`chooseServer()`函数，所以调用时会采用`BaseLoadBalancer`中的实现，即使用RoundRobinRule规则，通过线性轮询的方式来选择服务实例。由于这样并没有区域的概念，所以会将所有的节点都视为一个区域下的，这样就可能导致周期性的跨区域选择实例，因而会产生更高的延迟。而`ZoneAwareLoadBalancer`负载均衡器则可以解决这样的问题。

在`ZoneAwareLoadBalancer`中重写了函数`setServerListForZones(Map<String, List<Server>> zoneServersMap)`，此方法是在`DynamicServerListLoadBalancer`中定义的，我们可以看一下。

```java
protected void setServerListForZones(Map<String, List<Server>> zoneServersMap) {
    LOGGER.debug("Setting server list for zones: {}", zoneServersMap);
    this.getLoadBalancerStats().updateZoneServerMapping(zoneServersMap);
}
```

这个函数的调用地方是位于更新服务实例清单函数`setServersList()`的最后，且它的作用是根据区域分组的实例列表，为负载均衡器中的`LoadBalancerStats`对象创建`ZoneStats`并放入`Map zonsStatsMap`集合中，每个区域Zone对应一个ZonsStats，用来存储每个Zone的一些状态和统计信息。而`ZoneAwareLoadBalancer`对此方法的重写如下：

```java
private ConcurrentHashMap<String, BaseLoadBalancer> balancers = new ConcurrentHashMap();

protected void setServerListForZones(Map<String, List<Server>> zoneServersMap) {
    super.setServerListForZones(zoneServersMap);
  	//首先创建一个balancers对象，用来存储每个Zone区域对应的负载均衡器
    if (this.balancers == null) {
        this.balancers = new ConcurrentHashMap();
    }

    Iterator i$ = zoneServersMap.entrySet().iterator();

    Entry existingLBEntry;
    while(i$.hasNext()) {
        existingLBEntry = (Entry)i$.next();
        String zone = ((String)existingLBEntry.getKey()).toLowerCase();
      	//然后对于每个zone，通过调用getLoadBalancer来创建一个负载均衡器
      	//创建完成后再调用setServerList函数来设置Zone对应负载均衡器的实例清单
        this.getLoadBalancer(zone).setServersList((List)existingLBEntry.getValue());
    }

    i$ = this.balancers.entrySet().iterator();

  	//对Zone区域中实例清单的检查，看看是否有Zone区域下已没有实例了
  	//是的话就将balancers中对应Zone区域的实例列表清空
  	//这一步是为了防止过时的Zone区域统计信息干扰具体实例的选择算法
    while(i$.hasNext()) {
        existingLBEntry = (Entry)i$.next();
        if (!zoneServersMap.keySet().contains(existingLBEntry.getKey())) {
            ((BaseLoadBalancer)existingLBEntry.getValue()).setServersList(Collections.emptyList());
        }
    }

}


BaseLoadBalancer getLoadBalancer(String zone) {
    zone = zone.toLowerCase();
    BaseLoadBalancer loadBalancer = (BaseLoadBalancer)this.balancers.get(zone);
    if (loadBalancer == null) {  //这个zone还未创建负载均衡器
        IRule rule = this.cloneRule(this.getRule());
        loadBalancer = new BaseLoadBalancer(this.getName() + "_" + zone, rule, this.getLoadBalancerStats());
        BaseLoadBalancer prev = (BaseLoadBalancer)this.balancers.putIfAbsent(zone, loadBalancer);
        if (prev != null) {
            loadBalancer = prev;
        }
    }

    return loadBalancer;
}

private IRule cloneRule(IRule toClone) {
    Object rule;
    if (toClone == null) {
        rule = new AvailabilityFilteringRule();  //如果还未创建IRule实例，则新建一个
    } else {
        String ruleClass = toClone.getClass().getName();   //创建完了就克隆一个
        try {
            rule = (IRule)ClientFactory.instantiateInstanceWithClientConfig(ruleClass, this.getClientConfig());
        } catch (Exception var5) {
            throw new RuntimeException("Unexpected exception creating rule for ZoneAwareLoadBalancer", var5);
        }
    }

    return (IRule)rule;
}
```

最后我们再看看它是如何去挑选服务实例，来实现对区域的识别的：

```java
public Server chooseServer(Object key) {
  	//只有当负载均衡器中维护的实例所属的Zone区域个数大于1时，才会执行选择策略
  	//否则就只有一个Zone，无需针对区域进行选择，即还是父类的实现
    if (ENABLED.get() && this.getLoadBalancerStats().getAvailableZones().size() > 1) {
        Server server = null;

        try {
            LoadBalancerStats lbStats = this.getLoadBalancerStats();
          	//1. 为当前均衡负载器中所有的Zone区域分别创建快照 
            Map<String, ZoneSnapshot> zoneSnapshot = ZoneAvoidanceRule.createSnapshot(lbStats);
            if (this.triggeringLoad == null) {
                this.triggeringLoad = DynamicPropertyFactory.getInstance().getDoubleProperty("ZoneAwareNIWSDiscoveryLoadBalancer." + this.getName() + ".triggeringLoadPerServerThreshold", 0.2D);
            }

            if (this.triggeringBlackoutPercentage == null) {
                this.triggeringBlackoutPercentage = DynamicPropertyFactory.getInstance().getDoubleProperty("ZoneAwareNIWSDiscoveryLoadBalancer." + this.getName() + ".avoidZoneWithBlackoutPercetage", 0.99999D);
            }
						
          	//2. 获取可用的Zone区域集合，而这个可用是有一定条件的，如下
          	//2.1 剔除符合这些规则的Zone区域：实例数为0的Zone区域；Zone区域内实例的平均负载小于0，或者实例故障率>=阈值0.999
          	//2.2 根据Zone区域的实例平均负载计算出最差的Zone区域，即平均负载最高的Zone区域
          	//2.3 若在上面过程中没有符合剔除要求的区域，
            Set<String> availableZones = ZoneAvoidanceRule.getAvailableZones(zoneSnapshot, this.triggeringLoad.get(), this.triggeringBlackoutPercentage.get());
          	
          	//3. 获得可用的Zone区域集合不为null，且个数小于Zone区域总数，则随机选择一个
            if (availableZones != null && availableZones.size() < zoneSnapshot.keySet().size()) {
                String zone = ZoneAvoidanceRule.randomChooseZone(zoneSnapshot, availableZones);  //随机选
                if (zone != null) {
                  	//4. 选择好Zone区域后，再根据其对应的负载均衡器选择一个具体的实例
                    BaseLoadBalancer zoneLoadBalancer = this.getLoadBalancer(zone);
                    server = zoneLoadBalancer.chooseServer(key);
                }
            }
        } catch (Throwable var8) {
            logger.error("Unexpected exception when choosing server using zone aware logic", var8);
        }

        if (server != null) {
            return server;
        } else {
            logger.debug("Zone avoidance logic is not invoked.");
            return super.chooseServer(key);
        }
    } else {
        logger.debug("Zone aware logic disabled or there is only one zone");
        return super.chooseServer(key);
    }
}
```