nginx主要支持以下三种负载均衡策略：

- 轮询：也就是轮着将请求分发给各个服务器。
- 最少连接数：即将请求分发给拥有最少连接数的服务器
- ip-hash：根据客户端发送来的请求进行哈希计算，然后决定要使用哪个服务器

#### 默认策略

nginx中最简单的负载均衡配置如下：

```nginx
http {
    upstream myapp1 {
        server srv1.example.com;
        server srv2.example.com;
        server srv3.example.com;
    }

    server {
        listen 80;

        location / {
            proxy_pass http://myapp1;
        }
    }
}
```

上面配置假设有三个应用，当不设置负载均衡策略时默认就使用**轮询**的方式，所有的请求会被代理到myapp1上，然后nginx再针对请求将其分发到不同的服务器实例上。nginx的反向代理能对HTTP, HTTPS, FastCGI, uwsgi, SCGI, memcached, and gRPC等类型的请求进行负载均衡。例如将http改为https则会反向代理https请求。而其它的则需要使用类似[fastcgi_pass](http://nginx.org/en/docs/http/ngx_http_fastcgi_module.html#fastcgi_pass), [uwsgi_pass](http://nginx.org/en/docs/http/ngx_http_uwsgi_module.html#uwsgi_pass), [scgi_pass](http://nginx.org/en/docs/http/ngx_http_scgi_module.html#scgi_pass), [memcached_pass](http://nginx.org/en/docs/http/ngx_http_memcached_module.html#memcached_pass), 和 [grpc_pass](http://nginx.org/en/docs/http/ngx_http_grpc_module.html#grpc_pass) 指令。

#### 最少连接数

当有些请求需要很长时间才能完成时，此策略相对比较公平。当新请求到来时，nginx会尽量把它分发给连接数较少（没那么忙）的服务器上。此策略使用 [least_conn](http://nginx.org/en/docs/http/ngx_http_upstream_module.html#least_conn) 指令来实现，将指令放到**server group**内：

```nginx
upstream myapp1 {
    least_conn;
    server srv1.example.com;
    server srv2.example.com;
    server srv3.example.com;
}
```

#### session持久

如果使用轮询或者最少连接数策略，那么一个客户端的请求是无法保证每次都被分发到同一个服务器实例上的。如果需要让一个客户端来“绑定到”一个特定的实例上，可以使用ip-hash策略。它会将客户端的ip进行hash运算，并指向特定的服务器。只要服务器可用就始终会分发给此服务器。此策略使用 [ip_hash](http://nginx.org/en/docs/http/ngx_http_upstream_module.html#ip_hash) 指令来实现，放在**server group**内：

```nginx
upstream myapp1 {
    ip_hash;
    server srv1.example.com;
    server srv2.example.com;
    server srv3.example.com;
}
```

#### 权重负载均衡

上面的策略都是将所有的服务器实例同等对待，但也可以为每个服务器设置不同的权重。使用 [weight](http://nginx.org/en/docs/http/ngx_http_upstream_module.html#server) 参数，放在服务器的后面：

```nginx
upstream myapp1 {
    server srv1.example.com weight=3;
    server srv2.example.com;
    server srv3.example.com;
}
```

这种配置的意义表示：当有5个新的请求来到时，三个会分发给srv1，一个给srv2另一个给srv3。当然这主要针对的是默认（轮询）策略，而ip-hash和least-connected还没有开始。

#### 健康检查

nginx进行反向代理时会被动的检查服务器实例的健康情况，当某些服务器返回error不可用时，nginx会标记此服务器并尽量在一段时间内避免使用此服务器。

指令 [max_fails](http://nginx.org/en/docs/http/ngx_http_upstream_module.html#server) 设置了在 [fail_timeout](http://nginx.org/en/docs/http/ngx_http_upstream_module.html#server) 期间应与服务器进行连续不成功尝试通信的次数。max_fails默认为1，当设置为0时表明helth checks不可用。fail_timeout参数也定义了服务器被设置为failed的时间长度。服务器发生故障fail_timeout间隔后，nginx将针对客户端的请求来探测这个服务器，如果探测成功了就会将它标记为live。