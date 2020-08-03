---

name: Feign
title: 声明式服务调用：Feign
tags: 
categories:

date: 2020-06-07

---

**查看`Feign`的配置原理，在`FeignClientsRegistrar`类中去查看**。

***

Spring Cloud Feign整合了Spring Cloud Ribbon和Spring Cloud Hystrix，而且还提供了一种声明式的Web服务客户端定义方式。对于Spring Cloud Feign，我们只需要创建一个接口并用注解的方式来配置它，即可完成对服务提供方的接口绑定。

### 继承特性

能看到当使用Spring MVC的注解来绑定服务接口时，我们几乎可以完全从服务提供方的Controller中复制过来从而构建出相应的服务客户端绑定接口。而针对这些复制操作，Spring Cloud Feign提供了**继承特性**来帮助我们解决这些复制操作。我们看一下如何通过Spring Cloud Feign的继承特性来实现REST接口定义的复用。

### Ribbon配置

由于Spring Cloud Feign的客户端负载均衡是 通过Spring Cloud Ribbon实现的，所以我们可通过直接配置Ribbon客户端的方式来自定义各个服务客户端调用的参数。

#### 全局配置

全局配置的方式很简单，可直接使用`ribbon.<key>=<value>`的方式来设置Ribbon的各项默认参数，例如修改默认的客户端调用超时时间

```properties
ribbon.ConnectTimeout = 500
ribbon.ReadTimeout = 5000
```

#### 指定服务配置

大多数情况下，我们对于不同的服务会有不同的配置需求，而不是依赖统一的全局配置。在Spring Cloud Feign中针对各个服务，客户端进行个性化配置的方式和使用Spring Cloud Ribbon时的配置方式是一样的，都采用`<client>.ribbon.key = value`的格式。而这个`<client>`其实就是我们在使用`@FeignClient`注解时的`name`或者`value`属性。例如`@FeignClient(value = "HELLO-SERVICE")`那么此时我们就可以做如下的配置：

```properties
HELLO-SERVICE.ribbon.ConnectTimeout = 500
HELLO-SERVICE.ribbon.ReadTimeout = 500
HELLO-SERVICE.ribbon.OkToRetryOnAllOperations = true
HELLO-SERVICE.ribbon.MaxAutoRetriesNextServer = 2
HELLO-SERVICE.ribbon.MaxAutoRetries = 1
```

#### 重试机制

Spring Cloud Feign中默认实现了请求的重试机制，而上面的配置内容其实就已经定义了相关的配置。

### Hystrix配置

Spring Cloud Feign除了引入了用于客户端负载均衡的Spring Cloud Ribbon外，还引入了服务保护与容错的工具`Hystrix`，默认情况下，Spring Cloud Feign会为将所有`Feign`客户端的方法都封装到`Hystrix`进行服务保护。我们看一下在Spring Cloud Feign如何配置`Hystrix`属性以及如何实现服务降级。

#### 全局配置

和Spring Cloud Ribbon类似，直接使用它的默认配置前缀`hystrix.command.default`就可以进行设置，例如设置全局的超时时间：

```properties
hystrix.command.default.execution.isolation.thread.timeoutInMilliseconds = 5000
```

在对`Hystrix`进行配置之前，我们需要确认`feign.hystrix.enabled`参数有没有被设置为`false`，否则该参数配置会关闭`Feign`客户端的`Hystrix`支持。当然还可以通过`hystrix.command.default.execution.timeout.enabled = false`来关闭熔断功能。

#### 禁用Hystrix

如果只是想针对某个服务客户端关闭`Hystrix`支持时，需要通过使用`@Scope("prototype")`注解为指定的客户端配置`Feign.Builder`实例，详细步骤如下：

1. 构建一个关闭`Hystrix`的配置类

```java
@Configuration
public class DisableHystrixConfiguration {
		
		@Bean
		@Scope("prototype")
		public Feign.Builder feignBuilder() {
				return Feign.builder();
		}
}
```

2. 在`HelloService`的`@FeignClient`注解中，通过`configuration`参数引入上面实现的配置

```java
@FeignClient(name = "HELLO-SERVICE", configuration = DisableHystrixConfiguration.class)
public interface HelloService {
		...
}
```

#### 指定命令配置

对于`Hystrix`命令的配置，实际应用中也会有不同的配置方案。配置的方法和传统的`Hystrix`命令的参数配置相似，采样`hystrix.command.<commandKey>`作为前缀，而`<commandKey>`默认情况下会采用`Feign`客户端中的**方法名**作为标识。所以例如对于`/hello`接口的熔断超时时间的配置可通过其方法名作为`<commandKey>`来进行配置，如下：

```properties
hystrix.command.hello.execution.isolation.thread.timeoutInMillseconds = 5000
```

但因为会有方法重名的情况产生，所以这些方法会共用此配置。所以可以通过重写`Feign.Builder`的实现，并在应用主类中创建它的实例来覆盖自动化配置的`HystrixFeign.Builder`实现。

#### 服务降级配置

我们无法像之前Spring Cloud Hystrix中通过`@HystrixCommand`注解的`fallback`参数来指定具体的服务降级处理方法。但Spring Cloud Feign提供了另一种定义方式，

1. 需要为`Feign`客户端的定义接口编写一个具体的接口实现类。例如`HelloService`接口实现一个服务降级类`HelloServiceFallback`，**其中每个重写方法的实现逻辑都可以用来定义相应的服务降级逻辑**。

```java
@Component
public class HelloServiceFallback implements HelloService {

    @Override
    public String hello() {
        return null;
    }

    @Override
    public String hello(String name) {
        return null;
    }
}
```

2. 在服务绑定接口`HelloService`中，通过`@FeignClient`注解的`fallback`属性来指定对应的**服务降级实现类**。

```java
@FeignClient(name = "HELLO-SERVICE", fallback = HelloServiceFallback.class)  //指定服务降级实现类
public interface HelloService {
    @RequestMapping("/hello")
    String hello();

    @RequestMapping(value = "/hello1", method = RequestMethod.GET)
    String hello(@RequestParam("name") String name);
}
```

这样每个服务接口的断路器实际就是实现类中的重写函数的实现。

### 其他配置

#### 请求压缩

Spring Cloud Feign支持对请求与响应进行GZIP压缩，以减少通信过程中的性能损耗，我们只需要通过下面两个参数设置，就能开启请求与响应的压缩功能：

```properties
feign.compression.request.enabled = true
feign.compression.response.enabled = true
```

同时也可以对请求压缩做一些更细致的设置，例如指定压缩的请求数据类型，并设置了请求压缩的大小下限，只有超过这个大小的请求才会对其进行压缩。

#### 日志配置

Spring Cloud Feign在构建被`@FeignClient`注解修饰的服务客户端时，会为每个客户端创建一个`feign.Logger`实例，可以在`application.properties`文件中使用`logging.level.<FeignClient>`的参数配置格式来开启指定`Feign`客户端的DEBUG日志，其中`<FeignClient>`为`Feign`客户端定义接口的完整路径，例如：

```properties
logging.level.com.didispace.web.HelloService = DEBUG
```

但是只是添加如上配置还是无法输出DEBUG日志，因为`Feign`客户端默认的`Logger.level`对象定义为`NONE`级别，我们需要调整此级别。

针对于全局的日志级别，可以在**应用主类**中直接加入`Logger.Level`的Bean创建，如下：

```java
@EnableFeignClients
@EnableDiscoveryClient
@SpringBootApplication
public class ConsumerApplication {


    @Bean
    Logger.Level feignLoggerLevel() {
        return Logger.Level.INFO;   //这个地方没有FULL
    }

    public static void main(String[] args) {
        SpringApplication.run(ConsumerApplication.class, args);
    }
}
```

当然还可以通过实现配置类，然后在具体的`Feign`客户端来指定配置类以实现是否要调整不同的日志级别，例如：

```java
@Configuration
public class FullLogConfiguration {
    @Bean
    Logger.Level feignLoggerLevel() {
        return Logger.Level.INFO;
    }
}

@FeignClient(name = "HELLO-SERVICE", configuration = FullLogConfiguration.class)  //指定配置类
public interface HelloService1 {
    @RequestMapping("/hello")
    String hello();

    @RequestMapping(value = "/hello1", method = RequestMethod.GET)
    String hello(@RequestParam("name") String name);
}
```

对于`Feign`的`Logger`级别，主要有下面4类，可根据实际需要进行调整使用。

- NONE：不记录任何信息
- BASIC：仅记录请求方法、URL及响应状态码和执行时间
- HEADERS：处理记录BASIC级别的信息外，还会记录请求和响应的头信息
- FULL：记录所有请求与响应的明细，包括头信息、请求体、元数据等。

