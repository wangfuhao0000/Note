### 服务剖析

Nameko中一个service就是一个Python class，它将一些应用逻辑封装到了方法中，这些方法通过`entrypoint`暴漏给外部：

```python
from nameko.rpc import rpc, RpcProxy
class Service:
	name = "service"
    
	# we depend on the RPC interface of "another_service"
	other_rpc = RpcProxy("another_service")
    
	@rpc# `method` is exposed over RPC
	def method(self):
		# application logic goes here
		pass
```

#### Entrypoints

entrypoint是进入service的method的入口，它用来修饰这些method。这些entrypoint会监听一些事件（消息队列的或者http请求），然后执行对应的方法。

#### Dependencies

有些服务可能会依赖其它的部分，Nameko倾向于将那些逻辑作为dependencies。dependency通常不是service logic的一部分，而且它们的接口应尽可能简单，来让service进行调用。而且应该将dependencies看成连接service和其它代码（数据库、API）的网关。

#### Workers

当一个入口被触发时会创建workers。一个worker就是一个service类实例，它只是在一个方法执行时存在：服务从一个调用到另一个调用始终是**无状态的**。一个service能同时运行多个worker，这取决于用户的定义。

### 依赖注入

依赖注入就是向一个service类中声明一个属性，它是一个`DependencyProvider`，负责提供注入到service worker的实例。**一个worker的生命周期是**：

1. Entrypoint fires
2. Worker instantiated from service class
3. Dependencies injected into worker
4. Method executes
5. Worker is destroyed

反映到代码上类似下面的过程：

```
worker = Service()
worker.other_rpc = worker.other_rpc.get_dependency()
worker.method()
del worker
```

要注意dependency provider会在service存货期间一直存在，**但注入的实例对于每个worker来说是不同的**。

### 并发

Nameko是基于[eventlet](http://eventlet.net/)库搭建的，它通过"greenthreads"提供并发。并发性是具有隐式收益的协同例程。





### 扩展

所有的入口点



### 运行服务实例

当有了service类后，可以使用Nameko CLI来运行：

```shell
$ nameko run module:[ServiceClass]
```

此命令会查找指定module中的Nameko service实例并运行，当然也可以直接指定要运行的service类。

#### Service Containers

每个服务类都被委托给了`ServiceCOntainer`，这个容器封装了所有需要运行service的方法及需要的扩展。使用`ServiceContainer`运行单个service：

```python
from nameko.containers import ServiceContainer

class Service:
    name = "service"

# create a container
container = ServiceContainer(Service, config={})

# ``container.extensions`` exposes all extensions used by the service
service_extensions = list(container.extensions)

# start service
container.start()

# stop service
container.stop()
```

#### Service Runner

