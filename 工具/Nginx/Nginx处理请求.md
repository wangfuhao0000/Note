#### 基于命名的虚拟服务器

nginx首先决定使用哪个服务器处理新来的请求，假设有下面的配置包含三个虚拟服务器：

```nginx
server {
    listen      80;
    server_name example.org www.example.org;
    ...
}

server {
    listen      80;
    server_name example.net www.example.net;
    ...
}

server {
    listen      80;
    server_name example.com www.example.com;
    ...
}
```

首先nginx会根据请求头中的**"Host"**值来决定分发给哪个服务器，如果没有匹配的则会分发给监听此端口的**默认服务器**。一般默认服务器是第一个，当然可以使用`default_server`参数（在listen指令后）来指定自己是默认的服务器：

```nginx
server {
    listen      80 default_server;
    server_name example.net www.example.net;
    ...
}
```

> 注意参数`default_server`是监听端口的属性，而不是server name的属性。

#### 阻止带有未定义server name的请求

如果一个请求中没有"Host"头部，则nginx会拒绝这个请求：

```nginx
server {
    listen      80;
    server_name "";
    return      444;
}
```

设置server_name为""表示没有"Host"头部的请求，则返回nginx的444状态码，并关闭此连接。

#### name-based和IP-based的服务器

看一个更复杂的配置，一些虚拟服务器会监听不同的IP地址

```nginx
server {
    listen      192.168.1.1:80;
    server_name example.org www.example.org;
    ...
}

server {
    listen      192.168.1.1:80;
    server_name example.net www.example.net;
    ...
}

server {
    listen      192.168.1.2:80;  # ip和上面不一样
    server_name example.com www.example.com;
    ...
}
```

这个配置文件中，nginx首先根据每个server块的listen指令来测试请求的IP地址和端口。