我们知道，只需要在`RestTemplate`上加上一个`@LoadBalanced`注解就可以让`RestTemplate`自动实现客户端的负载均衡，我们一探究竟这到底是是如何实现的。



首先我们查看一个类`LoadBalancerClient`，它是Spring Cloud定义的一个接口，定义如下：

```java
public interface LoadBalancerClient {
  	//根据传入的服务名serviceId，从负载均衡中挑选一个对应服务的实例
    ServiceInstance choose(String serviceId);

  	//使用从负载均衡中挑选出的服务实例来执行请求内容
    <T> T execute(String serviceId, LoadBalancerRequest<T> request) throws IOException;

  	//为系统构建一个合适的host:port形式的URI
    URI reconstructURI(ServiceInstance var1, URI var2);
}
```

然后我们还能够找到一个类`LoadBalancerAutoConfiguration`，它是实现客户端负载均衡器的自动化配置类，查看一下源码。能看到这个配置类主要做了下面三个工作：

- 创建一个`LoadBalancerInterceptor`的Bean，用于实现对客户端发起请求时进行拦截，以实现客户端负载均衡
- 创建了一个`RestTemplateCustomizer`的Bean，用来给传进来的`RestTemplate`增加创建的拦截器
- 维护一个呗`@LoadBalancec`注解修饰的`RestTemplate`列表，并进行初始化。通过创建一个`SmartInitializingSingleton`的Bean来对里面的`RestTemplate`添加创建的拦截器。

```java
@Configuration
@ConditionalOnClass({RestTemplate.class})  //保证RestTemplate类存在于当前环境中
@ConditionalOnBean({LoadBalancerClient.class})   //Spring的Bean中必须有LoadBalancerClient
public class LoadBalancerAutoConfiguration {
    @LoadBalanced
    @Autowired(required = false)
    private List<RestTemplate> restTemplates = Collections.emptyList();

    public LoadBalancerAutoConfiguration() {
    }

    @Bean
    public SmartInitializingSingleton loadBalancedRestTemplateInitializer(final List<RestTemplateCustomizer> customizers) {
        return new SmartInitializingSingleton() {
            public void afterSingletonsInstantiated() {
                Iterator var1 = LoadBalancerAutoConfiguration.this.restTemplates.iterator();

                while(var1.hasNext()) {
                    RestTemplate restTemplate = (RestTemplate)var1.next();
                    Iterator var3 = customizers.iterator();

                    while(var3.hasNext()) {
                        RestTemplateCustomizer customizer = (RestTemplateCustomizer)var3.next();
                      	//对每个RestTemplate进行一个设置，其实就是添加创建的拦截器到它的拦截器列表中
                        customizer.customize(restTemplate);  
                    }
                }

            }
        };
    }

    @Bean
    @ConditionalOnMissingBean
    public RestTemplateCustomizer restTemplateCustomizer(final LoadBalancerInterceptor loadBalancerInterceptor) {
        return new RestTemplateCustomizer() {
            public void customize(RestTemplate restTemplate) {
              	//得到这个RestTemplate的所有拦截器，然后将下面创建的拦截器加入其中，最后重新设置新的拦截器列表
                List<ClientHttpRequestInterceptor> list = new ArrayList(restTemplate.getInterceptors());
                list.add(loadBalancerInterceptor);
                restTemplate.setInterceptors(list);
            }
        };
    }

    @Bean   //创建了一个拦截器LoadBalancerInterceptor，用于在客户端发起请求时，实现客户端负载均衡，参数是个客户端
    public LoadBalancerInterceptor ribbonInterceptor(LoadBalancerClient loadBalancerClient) {
        return new LoadBalancerInterceptor(loadBalancerClient);
    }
}
```

然后我们看看这个拦截器是如何让`RestTemplate`实现负载均衡的，具体要看这个拦截器的定义。能看到我们在拦截器中注入了`LoadBalancerClient`的实现，然后`RestTemplate`在进行HTTP请求时，会被拦截器的intercept方法所拦截，这个方法就会先会调用`LoadBalancerClient`的execute函数去根据服务名选择实例，并发起请求，而客户端的负载均衡也就是在这个execute函数中实现的。

```java
public class LoadBalancerInterceptor implements ClientHttpRequestInterceptor {
    private LoadBalancerClient loadBalancer;  //看上面，构造拦截器时会将客户端实现传进来

    public LoadBalancerInterceptor(LoadBalancerClient loadBalancer) {
        this.loadBalancer = loadBalancer;
    }
		
  	//拦截方法
    public ClientHttpResponse intercept(final HttpRequest request, final byte[] body, final ClientHttpRequestExecution execution) throws IOException {
        URI originalUri = request.getURI();
        String serviceName = originalUri.getHost();
      	//还记得我们刚才讲的接口里面有个execute方法，就是调用的客户端的execute方法来真正调用对应的服务
        return (ClientHttpResponse)this.loadBalancer.execute(serviceName, new LoadBalancerRequest<ClientHttpResponse>() {
            public ClientHttpResponse apply(ServiceInstance instance) throws Exception {
              	//构建那个服务Request
                HttpRequest serviceRequest = LoadBalancerInterceptor.this.new ServiceRequestWrapper(request, instance);
                return execution.execute(serviceRequest, body);
            }
        });
    }

  	//包装服务请求的Request
    private class ServiceRequestWrapper extends HttpRequestWrapper {
        private final ServiceInstance instance;   //服务实例，最后是下面的RibbonServer

        public ServiceRequestWrapper(HttpRequest request, ServiceInstance instance) {
            super(request);
            this.instance = instance;
        }

        public URI getURI() {   //利用客户端重构那个URI
            URI uri = LoadBalancerInterceptor.this.loadBalancer.reconstructURI(this.instance, this.
                                                                               

().getURI());
            return uri;
        }
    }
}
```

当然，目前我们只是使用的`LoadBalancerClient`接口，如果真正实现需要它的一个实现类，于是我们找到了一个类`RibbonLoadBalancerClient`，主要查看一下它的execute方法。

```java
public class RibbonLoadBalancerClient implements LoadBalancerClient {
    private SpringClientFactory clientFactory;

    public RibbonLoadBalancerClient(SpringClientFactory clientFactory) {
        this.clientFactory = clientFactory;
    }
		
    public URI reconstructURI(ServiceInstance instance, URI original) {
        Assert.notNull(instance, "instance can not be null");
        String serviceId = instance.getServiceId();
        RibbonLoadBalancerContext context = this.clientFactory.getLoadBalancerContext(serviceId);
        Server server = new Server(instance.getHost(), instance.getPort());
        boolean secure = this.isSecure(server, serviceId);
        URI uri = original;
        if (secure) {
            uri = UriComponentsBuilder.fromUri(original).scheme("https").build().toUri();
        }

        return context.reconstructURIWithServer(server, uri);
    }

    public ServiceInstance choose(String serviceId) {
        Server server = this.getServer(serviceId);
        return server == null ? null : new RibbonLoadBalancerClient.RibbonServer(serviceId, server, this.isSecure(server, serviceId), this.serverIntrospector(serviceId).getMetadata(server));
    }

    public <T> T execute(String serviceId, LoadBalancerRequest<T> request) throws IOException {
        ILoadBalancer loadBalancer = this.getLoadBalancer(serviceId);
        Server server = this.getServer(loadBalancer);
        if (server == null) {
            throw new IllegalStateException("No instances available for " + serviceId);
        } else {
          	//选择完服务器实例后构建一个RibbonServer，
            RibbonLoadBalancerClient.RibbonServer ribbonServer = new RibbonLoadBalancerClient.RibbonServer(serviceId, server, this.isSecure(server, serviceId), this.serverIntrospector(serviceId).getMetadata(server));
            RibbonLoadBalancerContext context = this.clientFactory.getLoadBalancerContext(serviceId);
            RibbonStatsRecorder statsRecorder = new RibbonStatsRecorder(context, server);

            try {
              	//调用request的apply方法，根据刚才创建的RibbonServer向一个具体的实例发起请求
              	//这个实例就是以host:port
                T returnVal = request.apply(ribbonServer);
                statsRecorder.recordStats(returnVal);  //将返回的结果记录下来
                return returnVal;
            } catch (IOException var9) {
                statsRecorder.recordStats(var9);
                throw var9;
            } catch (Exception var10) {
                statsRecorder.recordStats(var10);
                ReflectionUtils.rethrowRuntimeException(var10);
                return null;
            }
        }
    }

    private ServerIntrospector serverIntrospector(String serviceId) {
        ServerIntrospector serverIntrospector = (ServerIntrospector)this.clientFactory.getInstance(serviceId, ServerIntrospector.class);
        if (serverIntrospector == null) {
            serverIntrospector = new DefaultServerIntrospector();
        }

        return (ServerIntrospector)serverIntrospector;
    }

    private boolean isSecure(Server server, String serviceId) {
        IClientConfig config = this.clientFactory.getClientConfig(serviceId);
        if (config != null) {
            Boolean isSecure = (Boolean)config.get(CommonClientConfigKey.IsSecure);
            if (isSecure != null) {
                return isSecure;
            }
        }

        return this.serverIntrospector(serviceId).isSecure(server);
    }

    protected Server getServer(String serviceId) {
        return this.getServer(this.getLoadBalancer(serviceId));
    }
		
  	//真正的获取服务器实例的方法，调用的是ILoadBalancer.chooseServer实现的
    protected Server getServer(ILoadBalancer loadBalancer) {
        return loadBalancer == null ? null : loadBalancer.chooseServer("default");
    }

    protected ILoadBalancer getLoadBalancer(String serviceId) {
        return this.clientFactory.getLoadBalancer(serviceId);
    }
		
  	//对于服务实例的抽象定义，实现接口ServiceInstance
  	//不只包含Server对象，还存储了服务名，是否使用HTTPS标识，以及一个Map用于存储元数据
    protected static class RibbonServer implements ServiceInstance {
        private final String serviceId;
        private final Server server;
        private final boolean secure;  //是否使用HTTPS
        private Map<String, String> metadata;  //元数据

        //构造函数
      	//getter和setter
    }
}
```

我们能看到不管是哪个方法需要使用Server实例时调用的方法都是`protected Server getServer(ILoadBalancer loadBalancer)`这个方法，所以最终选择服务器实例的过程（负载均衡）也是由`ILoadBalancer`这个接口中的`chooseServer`方法去实现的。那我们先看一下这个接口的定义：

```java
public interface ILoadBalancer {
    void addServers(List<Server> var1);  //向负载均衡器维护的实例列表中添加服务器实例

    Server chooseServer(Object var1);   //通过某种策略从维护的实例中选择一个出来

    void markServerDown(Server var1);  //用来通知和标识某个服务器已经停止服务，防止此服务器被选择

    List<Server> getReachableServers();  //获取正常服务的实例列表

    List<Server> getAllServers();  //获取所有已知的服务实例列表
}
```

而通过`RibbonClientConfiguration`配置类我们知道接口`ILoadBalancer`默认的实现类是`ZoneAwareLoadBalancer`。

```java
@Bean
@ConditionalOnMissingBean
public ILoadBalancer ribbonLoadBalancer(IClientConfig config, ServerList<Server> serverList, ServerListFilter<Server> serverListFilter, IRule rule, IPing ping) {
  	//一系列的with方法，相当于一系列的set方法
    ZoneAwareLoadBalancer<Server> balancer = LoadBalancerBuilder.newBuilder().withClientConfig(config).withRule(rule).withPing(ping).withServerListFilter(serverListFilter).withDynamicServerList(serverList).buildDynamicServerListLoadBalancer();
    return balancer;
}
```