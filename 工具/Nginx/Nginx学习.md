nginx有一个`master`进程和多个`worker`进程：`master`进程用来读取配置和控制管理`worker`进程，而`worker`进程则用来处理一些请求。nginx使用基于事件的模型和依赖操作系统的机制来分配请求给不同的`worker`进程。`worker`进程的数量可以在配置文件中定义，当然也可以自适应。

默认的配置文件是`nginx.conf`，里面定义了nginx和它的moudle的工作方式。

## 开启，结束和重新加载配置

开启nginx就直接运行nginx命令即可，一旦开启就可以使用下面的命令控制它：

```shell
nginx -s signal
```

其中`signal`可以是如下参数：

- stop—快速关闭
- quit—优雅的关闭
- reload—重新加载配置文件，当更改配置文件后使用
- reopen—重新打开log文件

一旦`master`进程接受到要重新加载配置的信号，会检查新的配置文件语法是否有错，然后将更改的配置应用进来。

- 成功：**`master`进程会开启新的`worker`进程，并发送信号给老的`worker`进程告诉它们要终止。**
- 失败：`master`进程回滚配置的文件的更改，并继续使用老的配置。

当一个老的`worker`进程接受到终止信号时，**会停止接收新的连接并继续处理好当前的请求**，处理完成后老的`worker`进程会退出。

通常会使用Unix里面的一些命令（如`kill`）来对nginx的进程进行操作，通常操作的对象是一个确定的进程号PID。而nginx的`master`进程的PID是在文件`nginx.pid`里。想得到所有和nginx相关的进程信息可以使用`ps`命令：

```shell
ps -ax | grep nginx
```

## 配置文件的结构

nginx配置文件内包含了一些指令，通常有两种：**简单指令**和**块指令**。

- 简单指令通常包含name和parameters，用空格隔开并以分号`;`结尾
- 块指令和简单指令类似，只是它会用`{}`包起来。如果一个块指令能包含其它的块指令，那么它就叫做一个context（例如`events`，`http`， `server`，`location`）

在最外面的简单指令是在`main` context里，而`events`和`http`指令也在`main` context里，`server`在`http`里，`location`在`server`里。

## 提供静态内容

web服务器的一个功能就是提供文件，加入我们需要提供两个目录下的文件`data/www`和`data/images`，那么我们就需要**在http块下的server块内配置两个location块**。

首先在http块内写一个新的server块，通常一个http块会包含多个server块，并通过server names和监听的端口port来进行区分。一旦nginx决定让某个server来处理请求时，==它会测试内部定义的URI==。

```nginx
http {
	server {
	
	}
}
```

然后添加location块到server块内：

```nginx
location / {
	root /data/www;
}
```

location块指定了`"/"`前缀，通常来说会先选择最长的前缀进行匹配，最后才会匹配上面那种最短的情况。同样我们可以添加第二个location块：

```nginx
location /images/ {
	root /data;
}
```

上面这种配置就会匹配类似`/images/...`的URI，所以完整的server块如下：

```nginx
server {
    location / {
        root /data/www;
    }

    location /images/ {
        root /data;
    }
}
```

到此为止上面的配置就可以使用了，此时nginx监听端口80，在本地`http://localhost/`便可访问，且对于类似`http://localhost/images/example.png`这样的请求，会直接发送文件，不存在时发送404 error。而对于不是以`/images/`前缀开头的URI，则会匹配第一个即`/data/www`文件夹。

为了使用新的配置信息，可以使用`reload`指令发送给nginx的`master`进程：

```shell
nignx -s reload
```

如果程序和预期的不一样，可以在目录`/usr/local/nginx/logs`或者`/var/log/nginx`里面查看`access.log`和`error.log`。

## 设置一个简单的代理服务器

nginx一般用来设置为代理服务器，下面将设置一个代理服务器：接收对于images的请求，将其它的请求转发代理服务器，这里我们就使用一个nginx来实现。

首先在nginx配置文件中新建一个server块，它监听8080箭扣并把所有的请求都映射到对于本地目录`/data/up1`的访问上。注意到**root指令是在server context内的**，只是在location不包含自己的root指令时使用。

```nginx
server {
    listen 8080;
    root /data/up1;

    location / {
    }
}
```

然后使用上一节的配置信息并进行更改，使其变为一个代理服务器。手下你在第一个`location`块内，放一个`proxy_pass`指令并指定代理服务器的**协议，名称，端口**：

```nginx
server {
    location / {
        proxy_pass http://localhost:8080;
    }

    location /images/ {
        root /data;
    }
}
```

然后更改第二个`location`块，让其能接收传统的图像文件的访问：

```nginx
location ~ \.(gif|jpg|png)$ {
    root /data/images;
}
```

参数



## 改变配置

nginx可使用signals控制，通常主进程的ID会被写到文件`/usr/local/nginx/logs/nginx.pid`内。主进程能接收下面的一些信号：

```
TERM, INT	fast shutdown
QUIT	graceful shutdown
HUP	changing configuration, keeping up with a changed time zone (only for FreeBSD and Linux), starting new worker processes with a new configuration, graceful shutdown of old worker processes
USR1	re-opening log files
USR2	upgrading an executable file
WINCH	graceful shutdown of worker processes
```

单个的Worker进程也可以使用signals控制，它能接收的信号如下：

```
TERM, INT	fast shutdown
QUIT	graceful shutdown
USR1	re-opening log files
WINCH	abnormal termination for debugging (requires debug_points to be enabled)
```